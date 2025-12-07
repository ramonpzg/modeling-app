# Part 6: Error Handling Patterns in the Codebase

Error handling is one of Rust's strongest features and one of its steepest learning curves for Python programmers. Python uses exceptions; Rust uses types. This part examines how the KCL codebase handles errors, what patterns work, and where there's room for improvement.

## Rust's Error Philosophy

In Python, errors are exceptional events that interrupt normal flow:

```python
try:
    result = might_fail()
    process(result)
except SomeError as e:
    handle(e)
```

In Rust, errors are values that must be handled:

```rust
match might_fail() {
    Ok(result) => process(result),
    Err(e) => handle(e),
}
```

The key difference: Python lets you ignore errors (they'll crash at runtime). Rust forces you to acknowledge them (they're in the type signature).

## Result<T, E>: The Core Type

```rust
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result` is a generic enum. `T` is the success type; `E` is the error type. Every function that can fail returns `Result`.

### Using Result

```rust
fn parse_number(s: &str) -> Result<f64, ParseError> {
    s.parse().map_err(|_| ParseError::InvalidNumber(s.to_string()))
}

// Caller must handle both cases
let value = match parse_number("3.14") {
    Ok(n) => n,
    Err(e) => {
        eprintln!("Parse failed: {:?}", e);
        return Err(e.into());  // propagate
    }
};
```

### The ? Operator

Writing `match` everywhere gets tedious. The `?` operator propagates errors:

```rust
fn compile(code: &str) -> Result<Compiled, CompileError> {
    let tokens = lex(code)?;       // Returns early on Err
    let ast = parse(&tokens)?;     // Returns early on Err
    let checked = check(&ast)?;    // Returns early on Err
    Ok(generate(&checked))
}
```

The `?` operator:
1. If `Ok(value)`, unwraps to `value`
2. If `Err(e)`, returns `Err(e.into())` from the current function

This is like Python's implicit exception propagation, but explicit and type-checked.

## Option<T>: When Nothing Is Not an Error

```rust
pub enum Option<T> {
    Some(T),
    None,
}
```

`Option` represents "maybe a value." It's not for errors; it's for the absence of value.

### Option vs Result

| Use | Type | Example |
|-----|------|---------|
| Success or failure | `Result<T, E>` | Parsing, file I/O, network |
| Present or absent | `Option<T>` | HashMap lookup, optional field |

```rust
// Result: parsing can fail
fn parse(s: &str) -> Result<Value, ParseError>

// Option: key might not exist
fn get(&self, key: &str) -> Option<&Value>
```

### Converting Between Them

```rust
// Option to Result
let value = map.get("key")
    .ok_or_else(|| Error::KeyNotFound("key"))?;

// Result to Option
let maybe_value = parse("123").ok();  // discards error
```

## KCL's Error Types

Open `rust/kcl-error/src/lib.rs`. The main error type is:

```rust
#[derive(Debug, Clone)]
pub enum KclError {
    Lexical { details: KclErrorDetails },
    Syntax { details: KclErrorDetails },
    Semantic { details: KclErrorDetails },
    Type { details: KclErrorDetails },
    Argument { details: KclErrorDetails },
    UndefinedValue { details: KclErrorDetails, name: Option<String> },
    ValueAlreadyDefined { details: KclErrorDetails },
    Engine { details: KclErrorDetails },
    Internal { details: KclErrorDetails },
    // ... more variants
}

#[derive(Debug, Clone)]
pub struct KclErrorDetails {
    pub message: String,
    pub source_ranges: Vec<SourceRange>,
}
```

Each variant represents a category of error. The `KclErrorDetails` carries the message and source locations.

### Why an Enum?

Compare to Python, where you'd have multiple exception classes:

```python
class LexicalError(KclError): pass
class SyntaxError(KclError): pass
class TypeError(KclError): pass
```

Rust's enum approach has advantages:
1. Exhaustive matching: the compiler ensures you handle all error types
2. Single type: function signatures are simpler (`-> Result<T, KclError>`)
3. No heap allocation: enum variants are stored inline

### Creating Errors

The codebase uses constructor functions:

```rust
impl KclError {
    pub fn new_type(details: KclErrorDetails) -> Self {
        KclError::Type { details }
    }

    pub fn new_syntax(message: impl Into<String>, source_range: SourceRange) -> Self {
        KclError::Syntax {
            details: KclErrorDetails {
                message: message.into(),
                source_ranges: vec![source_range],
            },
        }
    }
}
```

Usage:

```rust
if expected_type != actual_type {
    return Err(KclError::new_type(KclErrorDetails::new(
        format!("Expected {}, got {}", expected_type, actual_type),
        vec![expr.source_range()],
    )));
}
```

## CompilationError: User-Facing Errors

`KclError` is for internal use. `CompilationError` is what users see:

```rust
pub struct CompilationError {
    pub source_range: SourceRange,
    pub message: String,
    pub severity: Severity,
    pub suggestion: Option<Suggestion>,
    pub tag: Tag,
}

pub enum Severity {
    Warning,
    Error,
    Fatal,
}

pub struct Suggestion {
    pub title: String,
    pub replacement: String,
    pub source_range: SourceRange,
}
```

### Severity Levels

- **Warning**: Something is suspicious but execution continues
- **Error**: Something is wrong; this part of execution fails
- **Fatal**: Unrecoverable; execution stops entirely

```rust
// Warning: deprecated feature
CompilationError::warning("startSketchAt is deprecated, use startSketchOn")
    .with_suggestion("Replace with", "startSketchOn", source_range)

// Error: type mismatch
CompilationError::err(source_range, "Expected Sketch, got Solid")

// Fatal: internal bug
CompilationError::fatal("Internal error: invalid state")
```

### Suggestions: Helping Users Fix Errors

The `Suggestion` system enables "quick fixes" in the IDE:

```rust
CompilationError::err(range, "Unknown variable 'lenght'")
    .with_suggestion(
        "Did you mean 'length'?",
        "length",
        range,
        Tag::Typo,
    )
```

In the editor, this appears as a clickable suggestion that replaces `lenght` with `length`.

## Error Propagation Patterns

The codebase uses several patterns for propagating errors.

### Pattern 1: Direct Propagation with ?

```rust
fn parse_expression(&mut self) -> Result<Expr, ParseError> {
    let left = self.parse_term()?;  // propagate if error
    let op = self.parse_operator()?;
    let right = self.parse_term()?;
    Ok(Expr::Binary { left, op, right })
}
```

Simple and clear. Works when the error type is already correct.

### Pattern 2: Mapping Errors

When you need to convert error types:

```rust
fn execute_file(&self, path: &Path) -> Result<ExecOutcome, ExecError> {
    let content = std::fs::read_to_string(path)
        .map_err(|e| ExecError::Io { source: e, path: path.to_owned() })?;

    let ast = parsing::parse_str(&content)
        .map_err(|errors| ExecError::Parse { errors })?;

    self.execute(&ast).await
}
```

`map_err` transforms the error type while propagating.

### Pattern 3: Collecting Multiple Errors

Sometimes you want to report all errors, not just the first:

```rust
fn check_all(items: &[Item]) -> Result<Vec<Checked>, Vec<CheckError>> {
    let mut results = Vec::new();
    let mut errors = Vec::new();

    for item in items {
        match check_one(item) {
            Ok(checked) => results.push(checked),
            Err(e) => errors.push(e),
        }
    }

    if errors.is_empty() {
        Ok(results)
    } else {
        Err(errors)
    }
}
```

KCL uses this pattern to show multiple parse errors at once instead of stopping at the first.

### Pattern 4: Non-Fatal Errors

`ExecOutcome` can contain both results and errors:

```rust
pub struct ExecOutcome {
    pub variables: IndexMap<String, KclValue>,
    pub errors: Vec<CompilationError>,
}
```

This allows partial execution: some things succeed, some fail, user sees both.

```rust
async fn execute(&mut self) -> ExecOutcome {
    let mut outcome = ExecOutcome::new();

    for item in &self.program.body {
        match self.execute_item(item).await {
            Ok(value) => outcome.variables.insert(item.name(), value),
            Err(e) => {
                outcome.errors.push(e.into());
                // Continue with next item
            }
        };
    }

    outcome
}
```

## The anyhow and thiserror Crates

The codebase uses two error-handling crates from the Rust ecosystem.

### thiserror: Custom Error Types

`thiserror` provides derive macros for error enums:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("Unexpected token: {0}")]
    UnexpectedToken(Token),

    #[error("Expected {expected}, got {actual}")]
    Mismatch { expected: String, actual: String },

    #[error("Unterminated string at position {0}")]
    UnterminatedString(usize),
}
```

The `#[error(...)]` attribute generates the `Display` implementation.

### anyhow: Quick and Dirty Errors

`anyhow` provides a type-erased error type for when you don't care about the specific error:

```rust
use anyhow::{Context, Result};

fn load_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.json")
        .context("Failed to read config file")?;

    serde_json::from_str(&content)
        .context("Failed to parse config JSON")
}
```

`anyhow::Result<T>` is shorthand for `Result<T, anyhow::Error>`, where `anyhow::Error` can hold any error type.

Use `anyhow` for:
- CLI tools
- Test code
- Prototyping
- When the caller doesn't need to match on error types

Use `thiserror` for:
- Library code
- When callers need to handle specific error cases
- When you want structured error data

## Error Handling Anti-Patterns

### Anti-Pattern 1: Panic on Recoverable Errors

```rust
// Bad: crashes on error
fn get_value(&self, key: &str) -> Value {
    self.map.get(key).unwrap()  // panics if key missing
}

// Good: returns Option or Result
fn get_value(&self, key: &str) -> Option<&Value> {
    self.map.get(key)
}
```

`unwrap()` and `expect()` should only be used when:
- You can prove the error is impossible
- It's test code
- Failure is truly unrecoverable

### Anti-Pattern 2: Stringly-Typed Errors

```rust
// Bad: loses type information
fn parse(s: &str) -> Result<Value, String> {
    Err("something went wrong".into())
}

// Good: structured error
fn parse(s: &str) -> Result<Value, ParseError> {
    Err(ParseError::InvalidSyntax { position: 42 })
}
```

String errors can't be matched on, can't carry structured data, and don't get compile-time checking.

### Anti-Pattern 3: Ignoring Errors

```rust
// Bad: silently ignores errors
let _ = might_fail();

// If you must ignore, document why
let _ = cleanup_temp_file();  // Ignore: best effort, not critical
```

The `let _ = ` pattern explicitly drops the result, silencing the "unused Result" warning. Sometimes this is correct, but it should be rare and documented.

### Anti-Pattern 4: Over-Catching

```rust
// Bad: catches everything, loses context
fn do_work() -> Result<(), Error> {
    parse()?;
    execute()?;
    output()?;
    Ok(())
}
// If execute() fails, which function was it?

// Better: add context
fn do_work() -> Result<(), Error> {
    parse().context("Failed to parse")?;
    execute().context("Failed to execute")?;
    output().context("Failed to output")?;
    Ok(())
}
```

## Error Display and Formatting

Look at how errors are formatted in `rust/kcl-error/src/lib.rs`:

```rust
impl std::fmt::Display for KclError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            KclError::Lexical { details } => {
                write!(f, "Lexical error: {}", details.message)
            }
            KclError::Syntax { details } => {
                write!(f, "Syntax error: {}", details.message)
            }
            // ...
        }
    }
}
```

For CLI output, errors are often rendered with source context:

```rust
fn render_error(error: &CompilationError, source: &str) -> String {
    let line = source.lines().nth(error.source_range.start_line).unwrap();
    let marker = " ".repeat(error.source_range.start_column) + "^";

    format!(
        "error: {}\n  --> {}:{}\n   |\n   | {}\n   | {}",
        error.message,
        error.file_name,
        error.source_range.start_line + 1,
        line,
        marker,
    )
}
```

Output:
```
error: Unexpected token
  --> main.kcl:3:15
   |
   | x = 5 ++ 3
   |       ^
```

## Exercises

1. **Error Chain**: Create a chain of functions where each can fail. Use `?` to propagate errors. Then use `.context()` from anyhow to add helpful messages at each level.

2. **Custom Error Type**: Define a new error type for a hypothetical KCL feature using `thiserror`. Include at least three variants with different data.

3. **Error Recovery**: Find a place in the parser that could recover from an error and continue parsing. Implement the recovery and collect multiple errors.

4. **Better Messages**: Find an error message in the codebase that could be more helpful. What information is missing? What would you add?

5. **Warning System**: Trace how warnings flow through the system. How do they differ from errors? When is a warning upgraded to an error?

## Common Mistakes and How to Fix Them

**Type mismatch when using ?**: The `?` operator calls `.into()` on the error. If there's no `From` implementation, you'll get a compile error.

```rust
// This fails if ParseError doesn't implement Into<ExecuteError>
fn execute_code(code: &str) -> Result<Value, ExecuteError> {
    let ast = parse(code)?;  // parse returns Result<_, ParseError>
    //                ^ error: `?` can't convert ParseError to ExecuteError
    execute(ast)
}

// Fix: add map_err
fn execute_code(code: &str) -> Result<Value, ExecuteError> {
    let ast = parse(code).map_err(ExecuteError::Parse)?;
    execute(ast)
}

// Or: implement From
impl From<ParseError> for ExecuteError {
    fn from(e: ParseError) -> Self {
        ExecuteError::Parse(e)
    }
}
```

**Unused Result warning**: Rust warns when you ignore a Result.

```rust
// Warning: unused result that must be used
write_file(path, content);

// Fix 1: handle the error
write_file(path, content)?;

// Fix 2: explicitly ignore (document why)
let _ = write_file(path, content);  // Best effort, non-critical
```

## What's Next

Part 7 covers testing strategies. Good error handling makes testing easier: you can verify that functions return the right errors for wrong inputs. Testing catches the bugs that error handling protects against.

Errors are inevitable. Good error handling makes them manageable.
