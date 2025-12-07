# Part 7: Testing Strategies

Tests are how you prove code works and keep it working. The KCL codebase uses several testing approaches: unit tests for functions, integration tests for pipelines, snapshot tests for outputs, and end-to-end tests that talk to real engines. This part covers each approach and when to use it.

## Rust's Testing Framework

Rust has testing built into the language. No pytest or unittest needed.

### Basic Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    fn test_divide_by_zero() {
        assert!(divide(1, 0).is_err());
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow_panics() {
        add(i32::MAX, 1);
    }
}
```

The `#[cfg(test)]` attribute means the module only compiles during testing. The `#[test]` attribute marks test functions.

### Running Tests

```bash
# All tests
cargo test

# Tests matching a pattern
cargo test parse

# Tests in a specific module
cargo test parsing::tests

# Show output (usually hidden)
cargo test -- --nocapture

# Single-threaded (for tests that interfere)
cargo test -- --test-threads=1
```

## Unit Tests in the Codebase

Unit tests live in the same file as the code they test, inside a `tests` module.

### Parser Unit Tests

Look at `rust/kcl-lib/src/parsing/parser.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_number_literal() {
        let tokens = tokenize("42").unwrap();
        let mut parser = Parser::new(&tokens);
        let expr = parser.parse_expression().unwrap();

        match &expr.inner {
            Expr::Literal(lit) => {
                assert!(matches!(lit.inner, LiteralValue::Number { value, .. } if value == 42.0));
            }
            _ => panic!("Expected number literal"),
        }
    }

    #[test]
    fn parse_binary_expression() {
        let tokens = tokenize("1 + 2 * 3").unwrap();
        let mut parser = Parser::new(&tokens);
        let expr = parser.parse_expression().unwrap();

        // 1 + (2 * 3) due to precedence
        match &expr.inner {
            Expr::BinaryExpression(bin) => {
                assert_eq!(bin.operator, BinaryOperator::Add);
                // Left is 1
                // Right is 2 * 3
            }
            _ => panic!("Expected binary expression"),
        }
    }
}
```

### Testing Error Cases

Good tests cover failure modes:

```rust
#[test]
fn parse_unterminated_string() {
    let result = tokenize("\"hello");
    assert!(result.is_err());

    let err = result.unwrap_err();
    assert!(err.iter().any(|e| e.message.contains("unterminated")));
}

#[test]
fn parse_invalid_operator() {
    let tokens = tokenize("1 @ 2").unwrap();
    let mut parser = Parser::new(&tokens);
    let result = parser.parse_expression();

    assert!(result.is_err());
}
```

## Integration Tests

Integration tests live in the `tests/` directory and test multiple components together.

### File Structure

```
rust/kcl-lib/
├── src/
│   └── ...
├── tests/
│   ├── parse_execute_test.rs
│   └── module_resolution_test.rs
└── Cargo.toml
```

### Example Integration Test

```rust
// tests/parse_execute_test.rs
use kcl_lib::execution::{execute, ExecutorContext};
use kcl_lib::parsing::parse_str;

#[tokio::test]
async fn execute_simple_assignment() {
    let code = r#"
        x = 5
        y = x + 3
    "#;

    let ast = parse_str(code).unwrap();
    let ctx = ExecutorContext::mock();
    let outcome = execute(&ast, &ctx).await.unwrap();

    assert_eq!(outcome.variables["x"], KclValue::number(5.0));
    assert_eq!(outcome.variables["y"], KclValue::number(8.0));
}

#[tokio::test]
async fn execute_function_call() {
    let code = r#"
        fn double(x) {
            return x * 2
        }
        result = double(21)
    "#;

    let ast = parse_str(code).unwrap();
    let ctx = ExecutorContext::mock();
    let outcome = execute(&ast, &ctx).await.unwrap();

    assert_eq!(outcome.variables["result"], KclValue::number(42.0));
}
```

### Async Tests

KCL execution is async. Tokio provides the test runtime:

```rust
#[tokio::test]
async fn async_test() {
    let result = some_async_function().await;
    assert!(result.is_ok());
}

// For more control over the runtime
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn multithreaded_test() {
    // Runs with multiple worker threads
}
```

## Snapshot Tests with Insta

Snapshot testing compares output against a stored reference. The KCL codebase uses [insta](https://insta.rs/) for this.

### How It Works

1. Run the test
2. Insta captures the output
3. First run: saves output as the snapshot
4. Later runs: compares output to snapshot
5. If different: test fails, you review the diff

### Parser Snapshot Tests

Look at `rust/kcl-lib/src/parsing/`:

```rust
use insta::assert_debug_snapshot;

#[test]
fn snapshot_parse_simple() {
    let ast = parse_str("x = 5 + 3").unwrap();
    assert_debug_snapshot!(ast);
}

#[test]
fn snapshot_parse_function() {
    let ast = parse_str(r#"
        fn add(a, b) {
            return a + b
        }
    "#).unwrap();
    assert_debug_snapshot!(ast);
}
```

The snapshot files are stored in `src/parsing/snapshots/`:

```
snapshots/
├── parsing__tests__snapshot_parse_simple.snap
├── parsing__tests__snapshot_parse_function.snap
└── ...
```

### Snapshot File Format

```
---
source: src/parsing/parser.rs
expression: ast
---
Node {
    inner: Program {
        body: [
            VariableDeclaration {
                name: Identifier("x"),
                value: BinaryExpression {
                    left: Literal(5),
                    operator: Add,
                    right: Literal(3),
                },
            },
        ],
    },
    start: 0,
    end: 9,
}
```

### Reviewing Snapshots

When a snapshot changes:

```bash
# Review all pending snapshots
cargo insta review

# Accept all changes (dangerous!)
cargo insta accept

# Reject all changes
cargo insta reject
```

The `cargo insta review` command opens an interactive UI where you can see diffs and accept/reject each change.

### When to Use Snapshots

Good for:
- AST structure (complex nested data)
- Error messages (exact wording matters)
- Generated output (code, configs)
- Serialized data (JSON, YAML)

Bad for:
- Floating-point values (precision issues)
- Random/time-dependent data
- Data with irrelevant ordering (sets, hashmaps)

## End-to-End Tests

E2E tests exercise the full pipeline, including the geometry engine.

### Test Structure

Look at `rust/kcl-lib/e2e/`:

```
e2e/
├── executor/
│   ├── inputs/
│   │   ├── cylinder/
│   │   │   └── input.kcl
│   │   ├── box_with_fillet/
│   │   │   └── input.kcl
│   │   └── ...
│   └── outputs/
│       ├── cylinder.snap
│       ├── cylinder.png
│       ├── box_with_fillet.snap
│       ├── box_with_fillet.png
│       └── ...
└── mod.rs
```

Each test:
1. Reads the input KCL file
2. Executes against the real engine (or mock)
3. Compares output to saved snapshot
4. Compares rendered image to saved PNG

### Running E2E Tests

```bash
# Requires engine connection
TWENTY_TWENTY=1 cargo test --test executor

# Without image comparison
cargo test --test executor
```

The `TWENTY_TWENTY` environment variable enables image comparison using the `twenty-twenty` crate.

### Creating New E2E Tests

The codebase has a helper:

```bash
just new-sim-test my_new_test
```

This creates:
- `e2e/executor/inputs/my_new_test/input.kcl`
- Empty snapshot files

Then you:
1. Write the KCL code in `input.kcl`
2. Run the test (it fails, creates new snapshot)
3. Review and accept the snapshot
4. Commit

## Mock Engine for Testing

Testing against a real engine is slow and requires network access. The mock engine enables fast, offline testing.

### MockEngineConnection

From `rust/kcl-lib/src/engine/conn_mock.rs`:

```rust
pub struct MockEngineConnection {
    responses: Arc<RwLock<IndexMap<Uuid, WebSocketResponse>>>,
}

impl MockEngineConnection {
    pub fn new() -> Self {
        Self {
            responses: Arc::new(RwLock::new(IndexMap::new())),
        }
    }

    pub fn with_response(mut self, cmd_id: Uuid, response: WebSocketResponse) -> Self {
        self.responses.write().insert(cmd_id, response);
        self
    }
}

impl EngineManager for MockEngineConnection {
    fn batch(&self) -> Result<Arc<RwLock<Vec<WebSocketRequest>>>, EngineError> {
        // Commands go nowhere
        Ok(Arc::new(RwLock::new(Vec::new())))
    }

    fn responses(&self) -> Result<Arc<RwLock<IndexMap<Uuid, WebSocketResponse>>>, EngineError> {
        // Return pre-configured responses
        Ok(self.responses.clone())
    }
}
```

### Using Mock in Tests

```rust
#[tokio::test]
async fn test_extrude_with_mock() {
    let code = r#"
        sketch = startSketchOn(XY)
          |> circle(center = [0, 0], radius = 10)
        solid = extrude(sketch, length = 5)
    "#;

    let mock = MockEngineConnection::new()
        .with_response(
            sketch_cmd_id,
            WebSocketResponse::StartSketch { sketch_id: Uuid::new_v4() },
        )
        .with_response(
            extrude_cmd_id,
            WebSocketResponse::Extrude {
                solid_id: Uuid::new_v4(),
                faces: vec![],
                edges: vec![],
            },
        );

    let ctx = ExecutorContext::with_engine(mock);
    let outcome = execute(&parse_str(code).unwrap(), &ctx).await.unwrap();

    assert!(outcome.variables.contains_key("solid"));
}
```

## Test Macros

The codebase defines custom test macros for common patterns.

### kcl_input! Macro

```rust
macro_rules! kcl_input {
    ($name:expr) => {
        include_str!(concat!("../e2e/executor/inputs/", $name, "/input.kcl"))
    };
}

#[test]
fn test_cylinder() {
    let code = kcl_input!("cylinder");
    // code is the contents of e2e/executor/inputs/cylinder/input.kcl
}
```

### Directory Test Macro

From `kcl-directory-test-macro`:

```rust
#[directory_test("e2e/executor/inputs")]
fn test_all_samples(path: &Path) {
    let code = std::fs::read_to_string(path.join("input.kcl")).unwrap();
    let result = parse_str(&code);
    assert!(result.is_ok(), "Failed to parse {:?}", path);
}
```

This generates a test for every directory in `e2e/executor/inputs/`.

## Property-Based Testing

For thorough testing, property-based testing generates random inputs.

### With proptest

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn number_parsing_roundtrip(n in -1e10..1e10f64) {
        let formatted = format!("{}", n);
        if let Ok(tokens) = tokenize(&formatted) {
            // If it tokenizes, it should parse
            let mut parser = Parser::new(&tokens);
            let result = parser.parse_expression();
            prop_assert!(result.is_ok() || formatted.contains("inf") || formatted.contains("nan"));
        }
    }

    #[test]
    fn identifier_valid_chars(s in "[a-zA-Z_][a-zA-Z0-9_]*") {
        let tokens = tokenize(&s).unwrap();
        assert_eq!(tokens.len(), 2);  // identifier + EOF
        assert!(matches!(tokens[0].token_type, TokenType::Identifier));
    }
}
```

Property testing finds edge cases you wouldn't think to write manually.

## Writing Effective Tests

### Test Names Matter

```rust
// Bad: what does this test?
#[test]
fn test1() { ... }

// Good: describes the scenario
#[test]
fn parse_binary_expression_respects_operator_precedence() { ... }

// Good: describes expected behavior
#[test]
fn division_by_zero_returns_error() { ... }
```

### Test One Thing

```rust
// Bad: tests multiple behaviors
#[test]
fn test_parser() {
    assert!(parse("1 + 2").is_ok());
    assert!(parse("fn x() {}").is_ok());
    assert!(parse("@settings").is_ok());
    assert!(parse("invalid @#$").is_err());
}

// Good: separate tests for separate behaviors
#[test]
fn parse_arithmetic_expression() { ... }

#[test]
fn parse_function_declaration() { ... }

#[test]
fn parse_annotation() { ... }

#[test]
fn reject_invalid_characters() { ... }
```

### Use Descriptive Assertions

```rust
// Bad: unhelpful failure message
assert!(result.is_ok());

// Good: explains what was expected
assert!(
    result.is_ok(),
    "Expected parsing to succeed, but got: {:?}",
    result.unwrap_err()
);

// Better: use assert_eq for equality
assert_eq!(
    result,
    Ok(expected),
    "Expected {:?} but got {:?}",
    expected,
    result
);
```

### Test Fixtures

For complex setup, use fixtures:

```rust
struct TestFixture {
    ctx: ExecutorContext,
    mock_engine: MockEngineConnection,
}

impl TestFixture {
    fn new() -> Self {
        let mock = MockEngineConnection::new();
        let ctx = ExecutorContext::with_engine(mock.clone());
        Self { ctx, mock_engine: mock }
    }

    async fn execute(&self, code: &str) -> ExecOutcome {
        let ast = parse_str(code).unwrap();
        execute(&ast, &self.ctx).await.unwrap()
    }
}

#[tokio::test]
async fn test_with_fixture() {
    let fixture = TestFixture::new();
    let outcome = fixture.execute("x = 5").await;
    assert_eq!(outcome.variables["x"], KclValue::number(5.0));
}
```

## Test Coverage

Rust has built-in coverage tools:

```bash
# Generate coverage report
cargo tarpaulin --out Html

# Open the report
open tarpaulin-report.html
```

Coverage shows which lines are executed by tests. It's a useful signal, not a goal. 100% coverage doesn't mean 100% correct.

## Exercises

1. **Add a Unit Test**: Pick a function in the parser that lacks tests. Write a test for its normal case and error case.

2. **Create a Snapshot Test**: Write a new KCL program and create a snapshot test for its AST. Intentionally break the parser and see what the diff looks like.

3. **Mock Engine Test**: Write a test that uses MockEngineConnection to test a specific KCL program without needing a real engine.

4. **Property Test**: Write a property test for some aspect of the lexer or parser. What edge cases does it find?

5. **E2E Test**: Create a new E2E test with a small KCL program. Run it against the mock engine. If you have access to a real engine, run it there too.

## What's Next

Part 8 covers the WASM bridge. Tests often need to work across the WASM boundary, and understanding how Rust and JavaScript communicate helps you write tests that work in both environments.

Tests are insurance. They cost something upfront but pay off when things break.
