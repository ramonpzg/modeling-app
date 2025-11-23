# Part 4: Traits, Generics, and KCL's Type System

You've built parsers. Now you'll understand how to write code that works with multiple types. Rust uses traits and generics for this. KCL uses traits extensively for its value system and standard library. By the end of this part, you'll understand how KCL tracks types at runtime and how Rust's compile-time type system makes this both fast and safe.

## Traits: Interfaces That Actually Work

If you've used Python's ABCs (Abstract Base Classes) or protocols, you've seen the idea:

```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    @abstractmethod
    def draw(self):
        pass

class Circle(Drawable):
    def draw(self):
        print("Drawing circle")

def render(obj: Drawable):
    obj.draw()
```

Python checks this at runtime (and only if you use type checkers). Rust checks it at compile time:

```rust
trait Drawable {
    fn draw(&self);
}

struct Circle {
    radius: f64,
}

impl Drawable for Circle {
    fn draw(&self) {
        println!("Drawing circle");
    }
}

fn render(obj: &impl Drawable) {
    obj.draw();
}
```

If `Circle` doesn't implement `draw`, the code won't compile. No runtime checks, no surprises.

## Generics: Code That Works for Any Type

Python has generics via type hints:

```python
from typing import TypeVar, Generic

T = TypeVar('T')

def first(items: list[T]) -> T:
    return items[0]

x = first([1, 2, 3])      # x is int
y = first(["a", "b"])      # y is str
```

But these are hints. The runtime doesn't enforce them. Rust's generics are real:

```rust
fn first<T>(items: &[T]) -> &T {
    &items[0]
}

let x = first(&[1, 2, 3]);       // x is &i32
let y = first(&["a", "b"]);      // y is &&str
```

The compiler generates separate code for each type. This is monomorphization. The generic function `first<T>` becomes `first_i32` and `first_str` at compile time. No runtime overhead.

You can constrain generics with trait bounds:

```rust
fn print_first<T: std::fmt::Display>(items: &[T]) {
    println!("{}", items[0]);
}
```

This says "T must implement the Display trait." If you try to use it with a type that doesn't implement Display, compilation fails.

## How KCL Uses Traits

KCL has a runtime type system. Values carry type information at runtime (unlike Rust, where types are compile-time only). But the implementation uses Rust's traits extensively.

Look at `rust/kcl-lib/src/execution/value.rs`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum KclValue {
    Uuid(Uuid),
    Bool(bool),
    Number(f64, Option<UnitType>),
    String(String),
    HomArray(Vec<KclValue>),
    Tuple(Vec<KclValue>),
    Object(HashMap<String, KclValue>),
    Function { /* ... */ },
    Sketch { /* ... */ },
    Solid { /* ... */ },
    // ... many more
}
```

Each KCL value is one of these variants. When you write `x = 5` in KCL, the runtime creates `KclValue::Number(5.0, None)`.

But how do you work with values generically? Traits.

## The `RuntimeType` Trait

KCL defines a trait for getting the type of a value:

```rust
pub trait RuntimeTyped {
    fn runtime_type(&self) -> RuntimeType;
}

impl RuntimeTyped for KclValue {
    fn runtime_type(&self) -> RuntimeType {
        match self {
            KclValue::Bool(_) => RuntimeType::Bool,
            KclValue::Number(_, unit) => RuntimeType::Number(*unit),
            KclValue::String(_) => RuntimeType::String,
            KclValue::HomArray(items) => {
                let elem_type = items.first()
                    .map(|v| v.runtime_type())
                    .unwrap_or(RuntimeType::Unknown);
                RuntimeType::Array(Box::new(elem_type))
            }
            // ... more cases
        }
    }
}
```

Any code that needs the type of a value can call `value.runtime_type()`. This works for all KCL values, regardless of their variant.

In Python:

```python
def runtime_type(value):
    if isinstance(value, bool):
        return "bool"
    elif isinstance(value, (int, float)):
        return "number"
    elif isinstance(value, str):
        return "string"
    # ... etc
```

But Python's version is runtime-only. Rust's version is compile-time verified to handle all cases (exhaustiveness checking on the match).

## Trait Objects: Dynamic Dispatch

Sometimes you want a collection of different types that implement the same trait:

```rust
trait Animal {
    fn make_sound(&self);
}

struct Dog;
impl Animal for Dog {
    fn make_sound(&self) {
        println!("Woof!");
    }
}

struct Cat;
impl Animal for Cat {
    fn make_sound(&self) {
        println!("Meow!");
    }
}

// This doesn't work (different types):
// let animals = vec![Dog, Cat];

// This works (trait objects):
let animals: Vec<Box<dyn Animal>> = vec![
    Box::new(Dog),
    Box::new(Cat),
];

for animal in &animals {
    animal.make_sound();
}
```

The `dyn Animal` is a trait object. It uses dynamic dispatch (like Python's method calls). The `Box<dyn Animal>` is a heap-allocated pointer to something that implements `Animal`.

This is slower than static dispatch (generic with monomorphization) but more flexible. You can mix different types in a single collection.

KCL uses trait objects for its function system. Functions can have different implementations (built-in, user-defined, stdlib), but they all implement a common trait.

## KCL's Type System: Runtime Types

Open `rust/kcl-lib/src/execution/types.rs`:

```rust
#[derive(Debug, Clone, PartialEq)]
pub enum RuntimeType {
    Bool,
    Number(Option<UnitType>),
    String,
    Array(Box<RuntimeType>),
    Tuple(Vec<RuntimeType>),
    Object(HashMap<String, RuntimeType>),
    Function {
        params: Vec<FunctionParam>,
        return_type: Box<RuntimeType>,
    },
    Sketch,
    Solid,
    // ... more variants
}
```

This is separate from Rust's type system. At compile time, everything is a `KclValue`. At runtime, each `KclValue` has a `RuntimeType`.

Why? KCL is dynamically typed. You can write:

```kcl
x = 5
x = "hello"  // Allowed (though unusual)
```

Rust wouldn't allow this. A Rust variable has one type, determined at compile time. But KCL needs runtime flexibility, so it implements its own type system on top of Rust's.

Here's how type checking works:

```rust
fn check_type(value: &KclValue, expected: &RuntimeType) -> Result<(), TypeError> {
    let actual = value.runtime_type();
    if actual == *expected {
        Ok(())
    } else {
        Err(TypeError::Mismatch {
            expected: expected.clone(),
            actual,
        })
    }
}
```

This is runtime type checking. Rust's compiler doesn't do this. KCL's executor does.

In Python, this is built-in:

```python
def check_type(value, expected_type):
    if not isinstance(value, expected_type):
        raise TypeError(f"Expected {expected_type}, got {type(value)}")
```

But Python checks at every operation. KCL checks only where necessary (function calls, type annotations), making it faster.

## Generics in Practice: KCL's Node Type

Remember `Node<T>` from the AST? Let's look at it again:

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

`Node<T>` works for any type `T`. You can have `Node<Expr>`, `Node<BodyItem>`, `Node<TypeDeclaration>`, etc. The compiler generates separate code for each, but you write the struct once.

This is how KCL attaches metadata (source positions, comments) to every AST node without duplicating code.

In Python, you'd use:

```python
@dataclass
class Node:
    inner: Any
    start: int
    end: int
    # ... etc

# Usage:
expr_node = Node(inner=some_expr, start=0, end=10)
```

But `Any` loses type information. Rust's `Node<T>` preserves it. The compiler knows `Node<Expr>` contains an `Expr`, not just "some value."

## Implementing Traits: Making Your Types Work

Let's build a simple type system for our mini-KCL:

```rust
#[derive(Debug, Clone, PartialEq)]
enum Type {
    Number,
    String,
    Bool,
    Array(Box<Type>),
    Function {
        params: Vec<Type>,
        return_type: Box<Type>,
    },
}

trait Typed {
    fn get_type(&self) -> Type;
}

#[derive(Debug, Clone)]
enum Value {
    Number(f64),
    String(String),
    Bool(bool),
    Array(Vec<Value>),
}

impl Typed for Value {
    fn get_type(&self) -> Type {
        match self {
            Value::Number(_) => Type::Number,
            Value::String(_) => Type::String,
            Value::Bool(_) => Type::Bool,
            Value::Array(items) => {
                let elem_type = items.first()
                    .map(|v| v.get_type())
                    .unwrap_or(Type::Number);  // Default to number
                Type::Array(Box::new(elem_type))
            }
        }
    }
}

fn type_check_binary_op(left: &Value, right: &Value, op: &str) -> Result<Type, String> {
    let left_type = left.get_type();
    let right_type = right.get_type();

    if left_type != right_type {
        return Err(format!("Type mismatch: {:?} {} {:?}", left_type, op, right_type));
    }

    match (left_type, op) {
        (Type::Number, "+") | (Type::Number, "-") | (Type::Number, "*") | (Type::Number, "/") => {
            Ok(Type::Number)
        }
        (Type::String, "+") => Ok(Type::String),  // String concatenation
        _ => Err(format!("Operator {} not supported for {:?}", op, left_type)),
    }
}
```

This gives you compile-time guarantees (all match cases covered) and runtime type checking (via `type_check_binary_op`).

## Where Clauses: Complex Trait Bounds

Sometimes trait bounds get complex:

```rust
fn process<T>(value: T)
where
    T: Clone + std::fmt::Debug + PartialEq,
{
    let copy = value.clone();
    println!("{:?}", copy);
    assert_eq!(value, copy);
}
```

The `where` clause lists all the trait bounds. This is equivalent to:

```rust
fn process<T: Clone + std::fmt::Debug + PartialEq>(value: T) {
    // ...
}
```

But `where` clauses are more readable when you have many bounds or complex types.

KCL's codebase uses `where` clauses extensively, especially in the execution engine where functions need to be `Send + Sync + 'static` (safe to send between threads and have no references with limited lifetimes).

## Associated Types: Traits That Return Types

Traits can have associated types:

```rust
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        self.count += 1;
        Some(self.count)
    }
}
```

The `Item` associated type says "whatever implements Iterator specifies what type of items it produces." For `Counter`, that's `u32`.

KCL uses associated types in its function trait:

```rust
trait KclFunction {
    type Output;
    fn call(&self, args: Vec<KclValue>) -> Result<Self::Output, ExecutionError>;
}
```

Different functions return different types (some return `KclValue`, some return `()`, etc.), but they all implement `KclFunction`.

## Building a Type Checker

Let's extend our mini-KCL with type checking:

```rust
struct TypeChecker {
    variables: HashMap<String, Type>,
}

impl TypeChecker {
    fn new() -> Self {
        TypeChecker {
            variables: HashMap::new(),
        }
    }

    fn check_stmt(&mut self, stmt: &Stmt) -> Result<(), String> {
        match stmt {
            Stmt::VarDecl(decl) => {
                let value_type = self.check_expr(&decl.value)?;
                self.variables.insert(decl.name.clone(), value_type);
                Ok(())
            }
            Stmt::Expr(expr) => {
                self.check_expr(expr)?;
                Ok(())
            }
        }
    }

    fn check_expr(&self, expr: &Expr) -> Result<Type, String> {
        match expr {
            Expr::Number(_) => Ok(Type::Number),
            Expr::String(_) => Ok(Type::String),
            Expr::Name(name) => {
                self.variables
                    .get(name)
                    .cloned()
                    .ok_or_else(|| format!("Undefined variable: {}", name))
            }
            Expr::Binary { left, op, right } => {
                let left_type = self.check_expr(left)?;
                let right_type = self.check_expr(right)?;

                if left_type != right_type {
                    return Err(format!(
                        "Type mismatch: {:?} and {:?}",
                        left_type, right_type
                    ));
                }

                match op {
                    BinOp::Add | BinOp::Sub | BinOp::Mul | BinOp::Div => {
                        if left_type == Type::Number {
                            Ok(Type::Number)
                        } else {
                            Err(format!("Arithmetic on non-numbers"))
                        }
                    }
                }
            }
        }
    }

    fn check_program(&mut self, program: &Program) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();

        for stmt in &program.statements {
            if let Err(e) = self.check_stmt(stmt) {
                errors.push(e);
            }
        }

        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
}
```

This checks types before execution, catching errors early.

## What You Just Learned

1. Traits are like interfaces but checked at compile time
2. Generics let you write code that works for multiple types (monomorphization)
3. Trait bounds constrain generics (`T: Trait`)
4. Trait objects (`dyn Trait`) allow dynamic dispatch and heterogeneous collections
5. KCL has runtime types separate from Rust's compile-time types
6. Associated types let traits specify related types
7. Where clauses make complex trait bounds readable
8. Type checkers walk ASTs and verify type correctness

## Exercises

1. **Implement Display for your types**: Add `impl std::fmt::Display for Value` to pretty-print values. Then use it in a REPL.

2. **Add function types**: Extend your type checker to handle function definitions and calls. Track function signatures in the symbol table.

3. **Generic AST walker**: Write a generic function `walk_ast<F>(node: &Expr, f: F)` where `F: Fn(&Expr)` that calls `f` on every node in the AST (including nested nodes).

4. **Explore KCL's types**: Read `rust/kcl-lib/src/execution/types.rs` and understand all the `RuntimeType` variants. Find where they're used in the executor.

5. **Type inference**: Extend your type checker to infer types from usage (like `let x = 5` doesn't need a type annotation because `5` is clearly a number).

## Next Up

In Part 5, we'll cover error handling in Rust and KCL. You'll understand `Result<T, E>`, `Option<T>`, the `?` operator, and how KCL's error types track source positions for helpful error messages. We'll also cover `panic!` (when to use it and when to avoid it) and how KCL recovers from errors during parsing and execution.

Errors are data, not exceptions.
