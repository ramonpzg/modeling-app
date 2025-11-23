# Part 2: Ownership, Borrowing, and Why KCL's AST Doesn't Leak Memory

You've seen Rust's ownership rules in theory. Now you'll see why they matter in practice. We're going to build a tool that parses KCL code and prints its abstract syntax tree (AST). Along the way, you'll understand why the KCL codebase is full of `Box<T>`, `Node<T>`, and `Vec<T>`, and why that's actually elegant.

## What Problem Does Ownership Solve?

In Python, this code is fine:

```python
def build_tree():
    root = {"type": "program", "children": []}
    child = {"type": "variable", "name": "x"}
    root["children"].append(child)
    return root

tree = build_tree()
# Use tree... Python's garbage collector eventually cleans it up
```

Python's reference counting keeps track of who's using what. When nothing points to `child` anymore, it gets freed. Simple. But reference counting has overhead. Every pointer change updates a counter. Cyclic references need a separate garbage collector. Memory usage is unpredictable.

Rust takes a different approach: every value has exactly one owner. When the owner goes away, the value is freed immediately. No counting, no garbage collection, no runtime overhead. But this means you have to think about ownership explicitly.

## The KCL AST Structure

Open `rust/kcl-lib/src/parsing/ast/types/mod.rs` and look at the `Node` type:

```rust
pub struct Node<T> {
    pub inner: T,
    pub start: usize,
    pub end: usize,
    pub module_id: ModuleId,
    pub outer_attrs: Vec<Node<Annotation>>,
    pub pre_comments: Vec<String>,
    pub comment_start: usize,
}
```

Every AST node wraps the actual data (`inner: T`) with metadata: source positions, module IDs, comments. This is like Python's AST, which also tracks line numbers and column offsets, but Rust makes the wrapper explicit.

Now look at how expressions are defined:

```rust
pub enum Expr {
    Literal(Box<Literal>),
    Name(Box<Name>),
    BinaryExpression(Box<BinaryExpression>),
    CallExpressionKw(Box<CallExpressionKw>),
    // ... many more variants
}
```

Notice the `Box<T>` everywhere? That's heap allocation. Let's understand why.

## Stack vs Heap: The Memory Layout Problem

In Rust, values are stored on the stack by default. The stack is fast but limited in size and requires knowing sizes at compile time. Consider this enum without boxes:

```rust
// This won't work (explained below)
enum BadExpr {
    Number(f64),                    // 8 bytes
    BinaryExpression(BinaryExpr),   // ??? bytes
}

struct BinaryExpr {
    left: BadExpr,   // ERROR: infinite size!
    right: BadExpr,  // ERROR: infinite size!
    operator: String,
}
```

The compiler can't determine the size of `BadExpr`. A `BinaryExpression` contains two `BadExpr` values, each of which might be another `BinaryExpression`, which contains two more `BadExpr` values, forever. Infinite size.

The solution: put the data on the heap and store a pointer on the stack:

```rust
enum Expr {
    Number(f64),                              // 8 bytes
    BinaryExpression(Box<BinaryExpression>),  // 8 bytes (pointer)
}

struct BinaryExpression {
    left: Expr,      // Pointer to heap
    right: Expr,     // Pointer to heap
    operator: String,
}
```

`Box<T>` is a heap-allocated pointer. It has a fixed size (one pointer), but points to data that can be any size. The compiler is happy.

In Python, everything is already heap-allocated and reference-counted, so you never think about this:

```python
class BinaryExpr:
    def __init__(self, left, right, operator):
        self.left = left    # Reference (implicitly heap-allocated)
        self.right = right   # Reference (implicitly heap-allocated)
        self.operator = operator
```

Rust makes you choose: stack (fast, limited) or heap (flexible, slightly slower). For recursive structures like ASTs, heap is required.

## Ownership in Action: Building an AST

Let's write code that uses KCL's parser. Create `learning_kcl/experiments/parse_kcl.rs`:

```rust
use kcl_lib::parsing::parse_str;

fn main() {
    let code = r#"
        width = 100
        height = 50
        area = width * height
    "#;

    // Parse the code
    match parse_str(code) {
        Ok(program) => {
            println!("Successfully parsed!");
            println!("Number of items in body: {}", program.body.len());

            // Print each item
            for item in &program.body {
                print_body_item(item);
            }
        }
        Err(errors) => {
            println!("Parse errors:");
            for error in errors {
                println!("  - {}", error);
            }
        }
    }
}

fn print_body_item(item: &BodyItem) {
    use BodyItem::*;
    match item {
        VariableDeclaration(decl) => {
            println!("Variable: {}", decl.inner.id.inner.name);
        }
        ExpressionStatement(_) => {
            println!("Expression statement");
        }
        _ => {
            println!("Other statement");
        }
    }
}
```

Wait, this won't compile yet. We need to import types. Fix it:

```rust
use kcl_lib::parsing::{parse_str, ast::types::BodyItem};

fn main() {
    // ... same as above
}

fn print_body_item(item: &BodyItem) {
    // ... same as above
}
```

Notice the `&` in `&program.body`? That's borrowing. We're iterating over references to items, not taking ownership of them. If we wrote `for item in program.body`, we'd move ownership into the loop, and `program` would be partially moved, which isn't allowed.

In Python:

```python
for item in program.body:  # Iterates over references (implicitly)
    print_body_item(item)
```

Python always uses references. Rust makes you choose: own or borrow?

## The Borrow Checker: Your New Best Friend (Who You'll Hate at First)

Try this modification:

```rust
fn main() {
    let code = "width = 100";
    let program = parse_str(code).unwrap();

    let first_item = &program.body[0];

    drop(program);  // Explicitly free program

    print_body_item(first_item);  // ERROR: borrowed value doesn't live long enough
}
```

The compiler stops you. `first_item` borrows from `program`, but you dropped `program` before using the borrow. Rust's borrow checker prevents use-after-free bugs at compile time.

Python would happily let you do this (and probably crash):

```python
program = parse_str(code)
first_item = program.body[0]
del program  # Might work, might crash, depends on reference counting
print(first_item)  # Undefined behavior if program was the last reference
```

Rust won't compile. Python will crash (sometimes). Which would you prefer at 3am when debugging production?

## Borrowing Rules: The Law of the Land

1. **You can have either one mutable reference OR any number of immutable references, but not both.**
2. **References must always be valid.**

Example:

```rust
let mut x = 5;

let r1 = &x;      // Immutable borrow
let r2 = &x;      // Another immutable borrow - fine
println!("{} {}", r1, r2);

let r3 = &mut x;  // Mutable borrow - ERROR: can't borrow as mutable because it's already borrowed as immutable
```

This prevents data races at compile time. If you could read and write simultaneously, you'd have a race condition.

In Python:

```python
x = [1, 2, 3]
r1 = x  # Reference
r2 = x  # Another reference
r1.append(4)  # Mutates through r1
print(r2)  # Sees the mutation through r2
```

Python allows simultaneous mutation. Rust doesn't. In single-threaded code, this is annoying. In multi-threaded code, this prevents race conditions.

## Building a Real AST Printer

Let's build something useful. Create `learning_kcl/experiments/ast_printer.rs`:

```rust
use kcl_lib::parsing::{parse_str, ast::types::{BodyItem, Expr, BinaryOperator}};

fn main() {
    let code = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "width = 100\nheight = 50".to_string());

    match parse_str(&code) {
        Ok(program) => {
            print_program(&program, 0);
        }
        Err(errors) => {
            eprintln!("Parse errors:");
            for error in errors {
                eprintln!("  {}", error);
            }
            std::process::exit(1);
        }
    }
}

fn print_program(program: &Program, indent: usize) {
    println!("{}Program", "  ".repeat(indent));
    for item in &program.body {
        print_body_item(item, indent + 1);
    }
}

fn print_body_item(item: &BodyItem, indent: usize) {
    let prefix = "  ".repeat(indent);
    match item {
        BodyItem::VariableDeclaration(decl) => {
            println!("{}VariableDeclaration: {}", prefix, decl.inner.id.inner.name);
            print_expr(&decl.inner.init.inner, indent + 1);
        }
        BodyItem::ExpressionStatement(expr) => {
            println!("{}ExpressionStatement", prefix);
            print_expr(&expr.inner, indent + 1);
        }
        _ => {
            println!("{}Other", prefix);
        }
    }
}

fn print_expr(expr: &Expr, indent: usize) {
    let prefix = "  ".repeat(indent);
    match expr {
        Expr::Literal(lit) => {
            println!("{}Literal: {:?}", prefix, lit.inner);
        }
        Expr::Name(name) => {
            println!("{}Name: {}", prefix, name.inner.name);
        }
        Expr::BinaryExpression(bin) => {
            println!("{}BinaryExpression: {:?}", prefix, bin.inner.operator);
            print_expr(&bin.inner.left.inner, indent + 1);
            print_expr(&bin.inner.right.inner, indent + 1);
        }
        _ => {
            println!("{}OtherExpression", prefix);
        }
    }
}
```

This won't compile as-is (we need more imports), but you get the idea. We're recursively walking the AST, borrowing at each level, never taking ownership. When the function returns, all borrows end, and the original owner (`program`) still owns everything.

## Lifetime Annotations: The Advanced Stuff (Preview)

Sometimes Rust can't figure out how long borrows should live. You help it with lifetime annotations:

```rust
fn first_item<'a>(program: &'a Program) -> &'a BodyItem {
    &program.body[0]
}
```

The `'a` is a lifetime parameter. It says "the returned reference lives as long as the input reference." This tells Rust that the returned borrow is valid as long as `program` is valid.

In Python, you'd write:

```python
def first_item(program):
    return program.body[0]  # Returns a reference (implicitly)
```

Python doesn't care about lifetimes. Rust makes you specify them when they're ambiguous.

KCL's AST uses lifetimes in a few places, but not extensively. Most of the time, ownership is clear enough that Rust can infer lifetimes automatically.

## Why This Matters for KCL

KCL's parser builds large ASTs. With Python's reference counting, every node allocation updates counters. Every tree traversal updates counters. With Rust's ownership, there's zero runtime overhead. Allocate once, traverse freely, drop the whole tree in one operation when you're done.

When KCL parses a complex program with thousands of nodes, Rust's ownership model makes it:
1. **Fast**: No reference counting overhead
2. **Safe**: No use-after-free bugs
3. **Predictable**: Memory is freed immediately when the owner goes out of scope

## What You Just Learned

1. `Box<T>` is heap allocation with ownership (required for recursive types)
2. Ownership prevents use-after-free bugs at compile time
3. Borrowing lets you reference data without taking ownership
4. The borrow checker enforces "one mutable reference OR many immutable references"
5. Stack is fast but limited; heap is flexible but requires pointers
6. Rust's ownership makes large ASTs faster than reference-counted equivalents
7. Lifetimes specify how long borrows are valid (usually inferred)

## Exercises

1. **Compile the AST printer**: Fix the imports and get `ast_printer.rs` working. You'll need to import `Program`, and possibly handle more expression types.

2. **Add more expression types**: Extend the printer to handle `CallExpressionKw`, `ArrayExpression`, and `ObjectExpression`. Check the AST types in `rust/kcl-lib/src/parsing/ast/types/mod.rs`.

3. **Write a node counter**: Write a function that counts the total number of nodes in a `Program`. It should recursively traverse the AST and return a count. Pay attention to borrowing.

4. **Ownership puzzle**: Write a function that takes ownership of a `Vec<String>`, filters it to only strings starting with "a", and returns a new `Vec<String>`. Then write a version that borrows the input and returns a new vector.

## Next Up

In Part 3, we'll look at pattern matching and how KCL's parser works. You'll understand how the `winnow` parser combinator library turns source code into ASTs, and you'll write a simple tokenizer for a subset of KCL. We'll also cover Rust's enums in depth, because KCL's AST is built on them.

The compiler's error messages are about to become your teacher.
