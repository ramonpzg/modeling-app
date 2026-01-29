# Part 05 — Errors, testing discipline, and validation

Rust forces you to model failure. Python lets you throw and hope. KCL wants explicit validation. Below are multiple examples per language to practice error handling and tests.

## Basic error handling
Python exceptions:
```python
def parse_int(text: str) -> int:
    return int(text)

try:
    parse_int("abc")
except ValueError as err:
    print("failed", err)
```
Rust `Result`:
```rust
fn parse_int(text: &str) -> Result<i32, std::num::ParseIntError> {
    text.parse::<i32>()
}

fn main() {
    match parse_int("abc") {
        Ok(v) => println!("{}", v),
        Err(e) => eprintln!("failed {}", e),
    }
}
```
KCL validation via `check`:
```kcl
schema Number:
    text: str
    value: int = int(text)

__main__:
    valid: Number = {text:"123"}
    # invalid: Number = {text:"abc"}  # type checker stops
    print(valid.value)
```
Python return Either style:
```python
from typing import Tuple, Optional
def safe_div(a: int, b: int) -> Tuple[Optional[int], Optional[str]]:
    if b == 0:
        return None, "division by zero"
    return a // b, None

print(safe_div(4,0))
```
Rust custom error enum:
```rust
#[derive(Debug)]
enum MathError { DivZero }

fn safe_div(a: i32, b: i32) -> Result<i32, MathError> {
    if b == 0 { return Err(MathError::DivZero); }
    Ok(a / b)
}

fn main() {
    println!("{:?}", safe_div(4,0));
}
```
KCL conditional expression:
```kcl
schema SafeDiv:
    a: int
    b: int
    result?: int = a / b if b != 0 else None
    error?: str = "division by zero" if b == 0 else None

case1: SafeDiv = {a:4, b:0}
__main__:
    print(case1.error)
```

## Testing basics
Python `unittest`:
```python
import unittest

class TestMath(unittest.TestCase):
    def test_add(self):
        self.assertEqual(1+1, 2)

if __name__ == '__main__':
    unittest.main()
```
Rust `#[test]`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_add() {
        assert_eq!(1 + 1, 2);
    }
}
```
KCL test with `kcl run` exit code expectations (conceptual):
```kcl
# test_add.k
schema Sum:
    a: int
    b: int
    total: int = a + b

assert Sum{a:1,b:1}.total == 2
```
Python pytest style:
```python
def test_upper():
    assert "kcl".upper() == "KCL"
```
Rust using `anyhow` for ergonomic errors:
```rust
use anyhow::{Result, anyhow};

fn might_fail(flag: bool) -> Result<()> {
    if flag { Ok(()) } else { Err(anyhow!("nope")) }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn test_fail() {
        assert!(might_fail(false).is_err());
    }
}
```
KCL validation with `check` block:
```kcl
schema Config:
    host: str
    port: int

config: Config = {host:"localhost", port:80}

check:
    assert config.port > 0
```

## Error propagation
Python chaining exceptions:
```python
def load_config(path: str) -> dict:
    import json
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError as e:
        raise RuntimeError("missing config") from e
```
Rust `?` operator:
```rust
use std::fs;
use std::error::Error;

fn load_config(path: &str) -> Result<String, Box<dyn Error>> {
    let data = fs::read_to_string(path)?;
    Ok(data)
}
```
KCL import failure idea:
```kcl
import "./nonexistent.k"  # would raise import error at compile time
```
Python contextmanager for cleanup:
```python
from contextlib import contextmanager
@contextmanager
def resource():
    print("open")
    try:
        yield "res"
    finally:
        print("close")

with resource() as r:
    print(r)
```
Rust RAII handles cleanup automatically:
```rust
struct Resource;
impl Drop for Resource {
    fn drop(&mut self) { println!("close"); }
}

fn main() {
    {
        let _r = Resource;
        println!("open");
    }
    println!("done");
}
```
KCL has no RAII, but recomputation is deterministic:
```kcl
schema Resource:
    state: str = "open"

use: Resource = {}

__main__:
    print(use.state)
```

## Property-based testing idea
Python hypothesis:
```python
from hypothesis import given, strategies as st
@given(st.lists(st.integers()))
def test_sorted(xs):
    assert sorted(xs) == list(sorted(xs))
```
Rust `proptest`:
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_sorted(mut xs: Vec<i32>) {
        let mut expected = xs.clone();
        expected.sort();
        xs.sort();
        prop_assert_eq!(xs, expected);
    }
}
```
KCL deterministic inputs; property tests are scenarios:
```kcl
schema Sorted:
    xs: [int]
    sorted: [int] = sorted(xs)

case: Sorted = {xs:[3,1,2]}

__main__:
    print(case.sorted)
```

## Exercises
1. For each error snippet, create a failing case and observe the message differences across languages.
2. Add logging to the Python and Rust examples; note how KCL lacks runtime logging and favors compile-time validation.
3. Write a Rust integration test that calls into `kcl-lib` functions (start with lexer), then describe how to mirror that validation in a KCL file.
4. Extend property testing to enforce idempotence of a formatting function in Python, Rust, and KCL.


## Extra code reps (errors and testing)

Python (4 blocks):
```python
# 1) try/except/else/finally
try:
    risky = 10 / 0
except ZeroDivisionError:
    risky = 0
else:
    print("no error")
finally:
    print("cleanup")
print(risky)
```
```python
# 2) pytest-style test
# file: test_math.py
import pytest

def add(a, b):
    return a + b

def test_add():
    assert add(1, 2) == 3
```
```python
# 3) contextlib for suppressing
from contextlib import suppress

with suppress(FileNotFoundError):
    open("missing.txt").read()
```
```python
# 4) custom exception
class KclError(Exception):
    pass

def parse(val):
    if not isinstance(val, str):
        raise KclError("expected string")
    return val.upper()

try:
    parse(1)
except KclError as exc:
    print(exc)
```

Rust (4 blocks):
```rust
// 1) Result handling with match
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("zero".into())
    } else {
        Ok(a / b)
    }
}
```
```rust
// 2) anyhow + thiserror style
use thiserror::Error;

#[derive(Error, Debug)]
pub enum KclError {
    #[error("parse failed: {0}")]
    Parse(String),
}
```
```rust
// 3) test module
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_divide() {
        assert_eq!(divide(4, 2).unwrap(), 2);
    }
}
```
```rust
// 4) Option chaining
fn parse_number(input: &str) -> Option<i32> {
    input.trim().parse::<i32>().ok()
}
```

KCL (4 blocks):
```kcl
# 1) try/except equivalent is `check`
val = check 10 / 2 else 0
print(val)
```
```kcl
# 2) Assertion in tests (using kcl test harness)
# file: tests/math.k
# assert add(1,2) == 3
```
```kcl
# 3) Option-like via union
parse_int = lambda s: int(s) if len(str(s)) > 0 else None
print(parse_int("42"))
```
```kcl
# 4) Custom error by contract
schema NonEmpty:
    value: str
    check len(value) > 0, "value required"

print(NonEmpty{value="ok"})
```
