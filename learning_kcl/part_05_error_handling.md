# Part 5: Error Handling Without Exceptions

Rust doesn't have exceptions. No `try`/`except`, no `raise`, no stack unwinding (by default). Errors are values. You return them, pattern match on them, and handle them explicitly. This feels alien at first. Then you realize you've stopped getting surprise exceptions at runtime.

KCL uses Rust's error handling extensively. When you write KCL code that fails to parse or execute, the errors come with source positions, helpful messages, and suggestions. Let's understand how that works.

## Result and Option: The Foundation

Python has exceptions:

```python
def divide(a, b):
    if b == 0:
        raise ValueError("Division by zero")
    return a / b

try:
    result = divide(10, 0)
except ValueError as e:
    print(f"Error: {e}")
```

Rust has `Result`:

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

match divide(10.0, 0.0) {
    Ok(result) => println!("Result: {}", result),
    Err(e) => println!("Error: {}", e),
}
```

`Result<T, E>` is an enum:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

A function returns either `Ok(value)` or `Err(error)`. The caller must handle both cases. The compiler enforces this. If you forget to check for errors, the code won't compile.

Python lets you ignore exceptions (they'll crash your program later). Rust makes you acknowledge them.

## The ? Operator: Error Propagation

In Python, exceptions propagate automatically:

```python
def parse_and_divide(text):
    # If int() fails, exception propagates to caller
    x = int(text)
    return divide(100, x)

# Caller must handle exception
try:
    result = parse_and_divide("0")
except (ValueError, ZeroDivisionError) as e:
    print(f"Error: {e}")
```

In Rust, you use `?`:

```rust
fn parse_and_divide(text: &str) -> Result<f64, String> {
    let x: f64 = text.parse()
        .map_err(|_| "Invalid number".to_string())?;
    divide(100.0, x)
}

match parse_and_divide("0") {
    Ok(result) => println!("Result: {}", result),
    Err(e) => println!("Error: {}", e),
}
```

The `?` operator:
1. If the result is `Ok(value)`, unwrap the value and continue
2. If the result is `Err(error)`, return early with the error

It's syntactic sugar for:

```rust
let x = match text.parse() {
    Ok(val) => val,
    Err(e) => return Err(e.to_string()),
};
```

The `?` operator makes error handling concise without hiding it. Python's exceptions are invisible until they explode. Rust's `?` is visible in the code.

## Option: The Maybe Monad (Without the Scary Name)

Python uses `None` for missing values:

```python
def find_user(id):
    users = {1: "Alice", 2: "Bob"}
    return users.get(id)  # Returns None if not found

user = find_user(3)
if user is not None:
    print(user)
else:
    print("User not found")
```

Rust uses `Option`:

```rust
fn find_user(id: u32) -> Option<&'static str> {
    let users = [(1, "Alice"), (2, "Bob")];
    users.iter()
        .find(|(uid, _)| *uid == id)
        .map(|(_, name)| *name)
}

match find_user(3) {
    Some(user) => println!("{}", user),
    None => println!("User not found"),
}
```

`Option<T>` is an enum:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

The compiler forces you to handle `None`. In Python, you might forget to check for `None` and get `AttributeError: 'NoneType' object has no attribute 'x'` at runtime. In Rust, that's a compile error.

You can use `?` with Option too:

```rust
fn get_first_user_name(users: &[User]) -> Option<String> {
    let first = users.first()?;  // Returns None if empty
    Some(first.name.clone())
}
```

If `users.first()` returns `None`, the function returns `None` early. Otherwise, it continues.

## KCL's Error Types

Open `rust/kcl-lib/src/errors.rs`:

```rust
#[derive(Debug, Clone)]
pub struct KclError {
    pub message: String,
    pub source_ranges: Vec<SourceRange>,
    pub error_type: KclErrorType,
}

#[derive(Debug, Clone)]
pub enum KclErrorType {
    Syntax,
    Semantic,
    Type,
    Runtime,
    Internal,
}

#[derive(Debug, Clone)]
pub struct SourceRange {
    pub start: usize,
    pub end: usize,
    pub module_id: ModuleId,
}
```

KCL errors carry:
1. A human-readable message
2. Source ranges (where the error occurred)
3. An error type (syntax, semantic, etc.)

This is much richer than Python's exceptions, which only have a message and traceback. KCL can show you exactly where in your code the error happened:

```
Error: Undefined variable 'x'
  --> box.kcl:5:10
   |
 5 |   area = x * height
   |          ^
```

This is possible because every AST node tracks its source position, and errors reference those positions.

## Implementing Custom Error Types

Let's build an error type for our mini-KCL:

```rust
#[derive(Debug, Clone)]
pub enum EvalError {
    UndefinedVariable {
        name: String,
        position: usize,
    },
    TypeMismatch {
        expected: String,
        actual: String,
        position: usize,
    },
    DivisionByZero {
        position: usize,
    },
}

impl std::fmt::Display for EvalError {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            EvalError::UndefinedVariable { name, position } => {
                write!(f, "Undefined variable '{}' at position {}", name, position)
            }
            EvalError::TypeMismatch { expected, actual, position } => {
                write!(
                    f,
                    "Type mismatch at position {}: expected {}, got {}",
                    position, expected, actual
                )
            }
            EvalError::DivisionByZero { position } => {
                write!(f, "Division by zero at position {}", position)
            }
        }
    }
}

impl std::error::Error for EvalError {}
```

Now you can use it:

```rust
fn evaluate(expr: &Expr, env: &HashMap<String, Value>) -> Result<Value, EvalError> {
    match expr {
        Expr::Number(n) => Ok(Value::Number(*n)),
        Expr::Name(name) => {
            env.get(name)
                .cloned()
                .ok_or_else(|| EvalError::UndefinedVariable {
                    name: name.clone(),
                    position: 0,  // You'd track this from the AST
                })
        }
        Expr::Binary { left, op, right } => {
            let left_val = evaluate(left, env)?;
            let right_val = evaluate(right, env)?;

            match (left_val, op, right_val) {
                (Value::Number(a), BinOp::Div, Value::Number(b)) => {
                    if b == 0.0 {
                        Err(EvalError::DivisionByZero { position: 0 })
                    } else {
                        Ok(Value::Number(a / b))
                    }
                }
                (Value::Number(a), BinOp::Add, Value::Number(b)) => {
                    Ok(Value::Number(a + b))
                }
                // ... more cases
                _ => Err(EvalError::TypeMismatch {
                    expected: "number".to_string(),
                    actual: "unknown".to_string(),
                    position: 0,
                }),
            }
        }
    }
}
```

The `?` operator propagates errors up the call stack. If any sub-expression fails, the whole evaluation fails. But unlike exceptions, you can see in the function signature that it returns `Result<Value, EvalError>`. The error handling is explicit.

## Multiple Error Types: The From Trait

Sometimes you have multiple error types. You can convert between them with `From`:

```rust
#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(ParseError),
    Eval(EvalError),
}

impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        AppError::Io(err)
    }
}

impl From<ParseError> for AppError {
    fn from(err: ParseError) -> Self {
        AppError::Parse(err)
    }
}

impl From<EvalError> for AppError {
    fn from(err: EvalError) -> Self {
        AppError::Eval(err)
    }
}

fn run_program(path: &str) -> Result<Value, AppError> {
    let code = std::fs::read_to_string(path)?;  // Converts io::Error to AppError
    let program = parse(&code)?;                 // Converts ParseError to AppError
    evaluate(&program)                           // Converts EvalError to AppError
}
```

The `?` operator automatically calls `From::from` to convert errors. This lets you unify different error types into a single error type for your function.

KCL uses this pattern extensively. Different parts of the system have different error types, and they're all converted to `KclError` at the top level.

## The anyhow and thiserror Crates

Most Rust projects use error handling libraries:

**`thiserror`** for library errors (explicit types):

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum EvalError {
    #[error("Undefined variable '{name}'")]
    UndefinedVariable { name: String },

    #[error("Type mismatch: expected {expected}, got {actual}")]
    TypeMismatch { expected: String, actual: String },

    #[error("Division by zero")]
    DivisionByZero,
}
```

The `#[error]` attribute generates the `Display` implementation automatically.

**`anyhow`** for application errors (dynamic):

```rust
use anyhow::{Context, Result};

fn run_program(path: &str) -> Result<Value> {
    let code = std::fs::read_to_string(path)
        .context("Failed to read input file")?;

    let program = parse(&code)
        .context("Failed to parse program")?;

    evaluate(&program)
        .context("Failed to evaluate program")
}
```

`anyhow::Result<T>` is an alias for `Result<T, anyhow::Error>`, where `anyhow::Error` can hold any error type. The `.context()` method adds context to errors without changing their type.

KCL doesn't use these libraries (it predates their widespread adoption), but you might use them in your Jupyter kernel.

## Panic: When Things Are Truly Unrecoverable

Sometimes errors are unrecoverable. The program is in an invalid state and can't continue:

```rust
let index = 10;
let items = vec![1, 2, 3];
let value = items[index];  // Panics: index out of bounds
```

`panic!` aborts the program (by default). It's like Python's `raise` without `try`/`except`:

```rust
fn must_be_positive(x: i32) {
    if x < 0 {
        panic!("Value must be positive, got {}", x);
    }
}
```

You use `panic!` for:
1. **Bugs** (invariant violations, internal errors)
2. **Impossible cases** (cases that should never happen)

You don't use `panic!` for:
1. **Expected errors** (user input errors, file not found, network errors)
2. **Anything recoverable**

KCL uses `panic!` sparingly, mostly for internal invariant violations. User errors are always `Result<T, E>`.

In Python, you might use assertions:

```python
assert x > 0, "x must be positive"
```

Rust has `assert!` too, but it's only for debugging:

```rust
assert!(x > 0, "x must be positive");
```

In release builds, assertions can be disabled. For production checks, use `Result`.

## Error Recovery: Collecting Multiple Errors

KCL's parser collects all errors instead of stopping at the first one. Here's a simplified version:

```rust
struct ParseContext {
    errors: Vec<ParseError>,
}

fn parse_program(input: &str) -> (Program, Vec<ParseError>) {
    let mut ctx = ParseContext { errors: Vec::new() };
    let mut statements = Vec::new();

    for line in input.lines() {
        match parse_statement(line, &mut ctx) {
            Ok(stmt) => statements.push(stmt),
            Err(e) => ctx.errors.push(e),
        }
    }

    let program = Program { statements };
    (program, ctx.errors)
}
```

This returns both the AST (possibly partial) and all errors. The caller can decide what to do: show all errors at once, or stop at the first error.

In Python, you'd typically use exceptions, which stop at the first error. To collect multiple errors, you'd need explicit accumulation:

```python
def parse_program(input):
    errors = []
    statements = []

    for line in input.splitlines():
        try:
            stmt = parse_statement(line)
            statements.append(stmt)
        except ParseError as e:
            errors.append(e)

    return Program(statements), errors
```

Rust's `Result` makes this pattern explicit and type-safe.

## Building an Error Reporter

Let's build a tool that shows errors with source context:

```rust
struct ErrorReporter<'a> {
    source: &'a str,
}

impl<'a> ErrorReporter<'a> {
    fn new(source: &'a str) -> Self {
        ErrorReporter { source }
    }

    fn report(&self, error: &EvalError) {
        let position = match error {
            EvalError::UndefinedVariable { position, .. } => *position,
            EvalError::TypeMismatch { position, .. } => *position,
            EvalError::DivisionByZero { position } => *position,
        };

        let (line_num, col_num) = self.position_to_line_col(position);

        eprintln!("Error: {}", error);
        eprintln!("  --> line {}:{}", line_num, col_num);

        if let Some(line) = self.get_line(line_num) {
            eprintln!("{:4} | {}", line_num, line);
            eprintln!("     | {}^", " ".repeat(col_num - 1));
        }
    }

    fn position_to_line_col(&self, position: usize) -> (usize, usize) {
        let mut line = 1;
        let mut col = 1;

        for (i, c) in self.source.chars().enumerate() {
            if i == position {
                break;
            }
            if c == '\n' {
                line += 1;
                col = 1;
            } else {
                col += 1;
            }
        }

        (line, col)
    }

    fn get_line(&self, line_num: usize) -> Option<&str> {
        self.source.lines().nth(line_num - 1)
    }
}
```

Usage:

```rust
let code = "width = 100\narea = x * width";  // x is undefined
let reporter = ErrorReporter::new(code);

match evaluate_program(code) {
    Ok(result) => println!("Result: {:?}", result),
    Err(errors) => {
        for error in &errors {
            reporter.report(error);
        }
    }
}
```

Output:

```
Error: Undefined variable 'x' at position 20
  --> line 2:8
   2 | area = x * width
     |        ^
```

This is how KCL shows errors with source context. The error type carries position information, and the reporter uses it to show the relevant code.

## What You Just Learned

1. Rust uses `Result<T, E>` instead of exceptions
2. `Option<T>` represents optional values (like Python's `None` but type-safe)
3. The `?` operator propagates errors explicitly
4. Custom error types carry rich information (messages, source positions, context)
5. `From` trait converts between error types
6. `panic!` is for unrecoverable errors (bugs, not user errors)
7. Error collection lets you report multiple errors at once
8. Error reporters use source positions to show context

## Exercises

1. **Improve error messages**: Add line and column numbers to your AST nodes. Update your error reporter to show them.

2. **Add error suggestions**: When a variable is undefined, search for similar variable names and suggest "Did you mean 'width'?" (use edit distance or simple string matching).

3. **Error recovery in parser**: Modify your parser to continue parsing after errors. When it encounters an unexpected token, skip to the next statement and continue.

4. **Test error cases**: Write test cases that deliberately trigger each error variant. Verify that they produce the expected error messages.

5. **Read KCL's errors**: Open `rust/kcl-lib/src/errors.rs` and understand all the error types. Find where they're created in the parser and executor.

## Next Up

In Part 6, we'll cover async Rust and KCL's execution model. You'll understand `async`/`await`, `Future`s, pinning, and how KCL communicates with the geometry engine asynchronously. We'll also cover the `tokio` runtime and how to write async functions that work with KCL's executor.

Async Rust is where things get weird. And powerful.
