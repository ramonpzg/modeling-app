# Part 1: Rust Fundamentals Through KCL's Parser and AST

You know Python. You want to learn Rust. The fastest way is through real code that does something you care about. KCL's parser turns text into tree structures, and understanding how it works teaches you Rust's core concepts: ownership, borrowing, lifetimes, and the type system that makes them work.

## The Python Mental Model (and Why Rust Breaks It)

In Python, you pass objects around without thinking about who owns them:

```python
def process_data(data):
    modified = transform(data)
    return modified

result = process_data(my_data)
# my_data is still valid, probably. Or maybe not. Who knows?
```

Python's garbage collector handles cleanup. You don't think about when memory gets freed because you don't have to. This convenience has a cost: you can't know at compile time whether `my_data` is still valid after that function call. You find out at runtime, often in production, usually at 3am.

Rust takes a different approach. It tracks ownership at compile time and refuses to build code that might access freed memory or create data races. The compiler is annoying until you understand it, then it becomes your most reliable colleague.

## Your First Look at Real Parser Code

Open `rust/kcl-lib/src/parsing/parser.rs`. This file is about 6,000 lines of recursive descent parsing. Don't read all of it. Let's look at a small piece:

```rust
// From rust/kcl-lib/src/parsing/parser.rs, lines 1-50 (simplified)
use crate::parsing::ast::types::*;
use crate::parsing::token::{Token, TokenType};

pub struct Parser<'a> {
    tokens: &'a [Token],
    current: usize,
}

impl<'a> Parser<'a> {
    pub fn new(tokens: &'a [Token]) -> Self {
        Parser { tokens, current: 0 }
    }

    fn peek(&self) -> Option<&Token> {
        self.tokens.get(self.current)
    }

    fn advance(&mut self) -> Option<&Token> {
        let token = self.tokens.get(self.current);
        self.current += 1;
        token
    }
}
```

Every line here teaches something about Rust. Let's unpack it.

## Ownership: Who Holds the Data

In Python, data exists somewhere in memory and you pass references around freely. In Rust, every piece of data has exactly one owner. When the owner goes out of scope, the data is freed.

```rust
fn main() {
    let tokens = vec![Token::new(...)];  // tokens owns this Vec
    let parser = Parser::new(&tokens);   // parser borrows tokens
}  // tokens is freed here
```

The `&` symbol creates a borrow. The parser doesn't own the tokens; it borrows them. This distinction matters because:

1. The tokens can't be freed while the parser is using them
2. The parser can't modify the tokens (it's an immutable borrow)
3. The compiler enforces both rules at compile time

Compare to Python:

```python
def main():
    tokens = [Token(...)]
    parser = Parser(tokens)
    # Who owns tokens now? Both main() and parser have references.
    # The garbage collector will figure it out... eventually.
```

## Lifetimes: How Long Can You Borrow

That `'a` syntax is a lifetime annotation. It says "the Parser struct lives for some lifetime called 'a, and the tokens it borrows must live at least that long."

```rust
pub struct Parser<'a> {
    tokens: &'a [Token],  // This reference must live as long as 'a
    current: usize,       // This owns its data, no lifetime needed
}
```

Why does Rust need this? Consider what happens without it:

```rust
fn broken_parser() -> Parser {  // Won't compile!
    let tokens = vec![Token::new(...)];
    Parser::new(&tokens)
}  // tokens is freed here, but Parser still references it!
```

The compiler sees that `tokens` would be freed before the returned `Parser`, and refuses to compile. The lifetime annotation makes this relationship explicit.

In Python, this bug exists but manifests differently:

```python
def broken_parser():
    tokens = [Token(...)]
    return Parser(tokens)
# Python's GC keeps tokens alive because Parser references it
# But if you're not careful with mutable state, surprises await
```

Python's garbage collector hides this complexity. Rust's lifetimes expose it. Neither approach is wrong; they're different tradeoffs.

## The impl Block: Methods on Types

```rust
impl<'a> Parser<'a> {
    pub fn new(tokens: &'a [Token]) -> Self {
        Parser { tokens, current: 0 }
    }
}
```

This is Rust's way of adding methods to a type. The `impl<'a>` says "this implementation is generic over lifetime 'a." The `Self` type refers to `Parser<'a>`.

Python equivalent:

```python
class Parser:
    def __init__(self, tokens):
        self.tokens = tokens
        self.current = 0
```

The Rust version is more explicit about what's happening. The Python version hides the same complexity behind dynamic typing.

## Borrowing in Methods: &self vs &mut self

```rust
fn peek(&self) -> Option<&Token> {
    self.tokens.get(self.current)
}

fn advance(&mut self) -> Option<&Token> {
    let token = self.tokens.get(self.current);
    self.current += 1;
    token
}
```

`&self` borrows the parser immutably. You can have many immutable borrows simultaneously.

`&mut self` borrows the parser mutably. You can only have one mutable borrow at a time, and no immutable borrows can exist simultaneously.

This is Rust's solution to data races: if you can mutate something, no one else can read it. If others are reading, you can't mutate.

```rust
let mut parser = Parser::new(&tokens);
let current = parser.peek();      // Immutable borrow
parser.advance();                  // Mutable borrow - would conflict!
println!("{:?}", current);         // Using the immutable borrow
```

The compiler catches this. In Python, you'd discover the bug when `current` mysteriously changes.

## Option: Rust's Null Alternative

```rust
fn peek(&self) -> Option<&Token> {
    self.tokens.get(self.current)
}
```

`Option<T>` is either `Some(value)` or `None`. It's how Rust handles the possibility of missing values without null pointer exceptions.

```rust
match parser.peek() {
    Some(token) => println!("Found: {:?}", token),
    None => println!("No more tokens"),
}

// Or more concisely:
if let Some(token) = parser.peek() {
    println!("Found: {:?}", token);
}
```

Python equivalent with Optional typing:

```python
from typing import Optional

def peek(self) -> Optional[Token]:
    if self.current < len(self.tokens):
        return self.tokens[self.current]
    return None
```

The difference: Python's Optional is a hint. Rust's Option is enforced. You can't use `Option<T>` as if it were `T` without explicitly handling the `None` case.

## The AST: Where Parsed Code Lives

After parsing, KCL code becomes an Abstract Syntax Tree. Look at `rust/kcl-lib/src/parsing/ast/types/mod.rs`:

```rust
// Simplified from the actual code
#[derive(Debug, Clone)]
pub struct Node<T> {
    pub inner: T,
    pub start: usize,
    pub end: usize,
    pub module_id: ModuleId,
}

#[derive(Debug, Clone)]
pub enum Expr {
    Literal(Node<LiteralValue>),
    Identifier(Node<Identifier>),
    BinaryExpression(Node<BinaryExpression>),
    CallExpression(Node<CallExpression>),
    PipeExpression(Node<PipeExpression>),
    // ... many more variants
}
```

### Generic Types: Node<T>

`Node<T>` is a generic wrapper that adds source location to any type. The `T` can be anything: a literal, an identifier, an expression.

```rust
let literal_node: Node<LiteralValue> = Node {
    inner: LiteralValue::Number(42.0),
    start: 0,
    end: 2,
    module_id: ModuleId::default(),
};
```

Python equivalent:

```python
from dataclasses import dataclass
from typing import TypeVar, Generic

T = TypeVar('T')

@dataclass
class Node(Generic[T]):
    inner: T
    start: int
    end: int
    module_id: ModuleId
```

### Enums: More Than Python's Enum

Rust's enums can hold data. `Expr` isn't just a list of names; each variant carries its associated data:

```rust
pub enum Expr {
    Literal(Node<LiteralValue>),      // Contains a Node<LiteralValue>
    BinaryExpression(Node<BinaryExpression>),  // Contains BinaryExpression data
    // ...
}
```

Python's closest equivalent uses Union types:

```python
from typing import Union

Expr = Union[
    Node[LiteralValue],
    Node[BinaryExpression],
    # ...
]
```

But Rust's version is better because:
1. The variants are named (you know it's a `Literal`, not just some `Node<LiteralValue>`)
2. Pattern matching is exhaustive (the compiler ensures you handle all cases)
3. The type is closed (no one can add variants later and break your code)

### Pattern Matching: Destructuring with match

```rust
fn eval_expr(expr: &Expr) -> Value {
    match expr {
        Expr::Literal(node) => eval_literal(&node.inner),
        Expr::BinaryExpression(node) => {
            let left = eval_expr(&node.inner.left);
            let right = eval_expr(&node.inner.right);
            apply_operator(node.inner.operator, left, right)
        }
        Expr::Identifier(node) => lookup_variable(&node.inner.name),
        // The compiler forces you to handle all variants
    }
}
```

If you add a new `Expr` variant, the compiler flags every `match` that doesn't handle it. In Python, you'd use isinstance checks and hope you remembered all the cases:

```python
def eval_expr(expr):
    if isinstance(expr.inner, LiteralValue):
        return eval_literal(expr.inner)
    elif isinstance(expr.inner, BinaryExpression):
        left = eval_expr(expr.inner.left)
        right = eval_expr(expr.inner.right)
        return apply_operator(expr.inner.operator, left, right)
    # Forgot a case? Runtime error awaits.
```

## Derive Macros: Automatic Trait Implementation

```rust
#[derive(Debug, Clone)]
pub struct Node<T> {
    // ...
}
```

`#[derive(Debug, Clone)]` automatically implements two traits:
- `Debug`: Enables `println!("{:?}", node)` for debugging output
- `Clone`: Enables `node.clone()` to create a copy

In Python, you get similar behavior for free (everything has `__repr__` and is copyable by default). Rust makes you opt in, which means the compiler knows exactly what each type supports.

Other common derives in the codebase:
- `Serialize, Deserialize`: JSON/binary serialization (from serde)
- `PartialEq, Eq`: Equality comparison
- `Hash`: Hashable for use in HashMaps
- `Default`: Creates a default instance

## The Parse Function: Putting It Together

Look at how parsing starts in `rust/kcl-lib/src/parsing/mod.rs`:

```rust
pub fn parse_str(input: &str) -> Result<Node<Program>, Vec<CompilationError>> {
    let tokens = tokeniser::lex(input)?;
    let mut parser = Parser::new(&tokens);
    parser.parse_program()
}
```

### Result: Errors as Values

`Result<T, E>` is either `Ok(value)` or `Err(error)`. Unlike exceptions, errors are part of the function's return type. You can't ignore them.

```rust
match parse_str(code) {
    Ok(ast) => execute(ast),
    Err(errors) => report_errors(errors),
}

// The ? operator propagates errors
fn compile(code: &str) -> Result<Compiled, Error> {
    let ast = parse_str(code)?;  // Returns early if parsing fails
    let analyzed = analyze(ast)?;
    let generated = codegen(analyzed)?;
    Ok(generated)
}
```

Python equivalent with exceptions:

```python
def compile(code):
    try:
        ast = parse_str(code)
        analyzed = analyze(ast)
        generated = codegen(analyzed)
        return generated
    except CompilationError as e:
        raise
```

Rust's version is more explicit. You know which functions can fail by looking at their signatures. Python's version is shorter but hides the failure modes.

## Memory Layout: Why This Matters

Rust's ownership system isn't arbitrary. It enables zero-cost abstractions.

When you have `Vec<Node<Expr>>` (a vector of AST nodes), the memory layout is:
- A contiguous block for the Vec's metadata (pointer, length, capacity)
- A contiguous block for the Node structs
- Each Node contains its inner data inline (no pointer indirection for simple types)

In Python, `list[Node]` is:
- A list object with its own overhead
- Pointers to each Node object
- Each Node is a separate heap allocation
- Each field in Node is another pointer to a Python object

Rust's layout is cache-friendly and predictable. The parser can iterate through tokens without chasing pointers all over the heap.

## Exercises

1. **Read the Tokenizer**: Open `rust/kcl-lib/src/parsing/token/tokeniser.rs`. Find where the pipe operator `|>` is tokenized. What happens if you type `| >` with a space?

2. **Trace a Parse**: Add this KCL code to a test and trace how it's parsed:
   ```kcl
   x = 5 + 3
   ```
   What sequence of parser methods gets called? What does the resulting AST look like?

3. **Find a Lifetime Bug**: Try to write Rust code that creates a dangling reference. What error does the compiler give? How does the lifetime annotation in the error message relate to the code?

4. **Compare Error Handling**: Find three places in the parser that return `Result`. How do they handle errors differently? When do they use `?` vs explicit `match`?

## What's Next

You've seen how ownership, borrowing, and lifetimes work in real parsing code. Part 2 explores KCL's type system and how Rust's memory model shaped the language's design decisions. The constraints that feel restrictive now become the foundation for powerful abstractions later.

The parser is just the beginning. Code becomes trees, trees become values, values become geometry. The chain from text to physical object starts here.
