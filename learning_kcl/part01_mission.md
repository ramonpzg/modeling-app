# Part 01 — Mission, map, and day-one reps

Welcome to the lab. You already speak Python. We will weaponize that to learn Rust and KCL together while steering toward a first contribution: a Jupyter kernel for KCL. This document lays out the contract: fast feedback, hands-on first, theory only when it unblocks the next keystroke. Expect sarcasm when you stall.

## Ground rules
- Build from day zero: clone, build, run tests, read code, then write code.
- Translate every new Rust move into a Python mental model, then re-express it in KCL.
- Keep a notebook of friction points; those become issues or docs fixes in this repo.
- No passive reading: every concept gets a code rep in all three languages.

## Repository orientation
- `rust/kcl-lib`: core language engine (lexer, parser, evaluator, type system). This is where you will read and later edit.
- `learning_kcl`: these tutorials. Edit them when you find holes.
- `packages/kcl-cli`: command-line wrapper; good for smoke testing.
- `packages/kcl-wasm`: WASM target; useful mental model for kernel messaging.
- `docs/` and `openapi/`: external surface area.

## Quick setup
- Install Rust (rustup), Node (for CLI tooling), and `just`/`make` if you want the shortcuts.
- Build check: `cd rust/kcl-lib && cargo test`.
- Run CLI: `node packages/kcl-cli/dist/index.js --help` after `npm install`.

## Code warmup: value binding
Python baseline:
```python
x = 3
name = "kcl"
message = f"{name} -> {x * 2}"
print(message)
```
Rust mirror:
```rust
fn main() {
    let x: i32 = 3;
    let name = "kcl";
    let message = format!("{} -> {}", name, x * 2);
    println!("{}", message);
}
```
KCL flavor:
```kcl
schema Main:
    x: int = 3
    name: str = "kcl"
    message: str = "${name} -> ${x * 2}"

main: Main {
    print(message)
}
```
Python with small refactor for clarity:
```python
def double_and_tag(x: int, tag: str) -> str:
    return f"{tag}:{x * 2}"

print(double_and_tag(5, "kcl"))
```
Rust variant with function:
```rust
fn double_and_tag(x: i32, tag: &str) -> String {
    format!("{}:{}", tag, x * 2)
}

fn main() {
    println!("{}", double_and_tag(5, "kcl"));
}
```
KCL variant using a function-like schema:
```kcl
schema Tagger:
    x: int
    tag: str
    result: str = "${tag}:${x * 2}"

item: Tagger {x:5, tag:"kcl"}

__main__:
    print(item.result)
```

## Code warmup: control and iteration
Python loop:
```python
for i in range(3):
    print(i * i)
```
Rust loop:
```rust
fn main() {
    for i in 0..3 {
        println!("{}", i * i);
    }
}
```
KCL comprehension-like expression:
```kcl
nums: [int] = [0,1,2]
squares = [n*n for n in nums]

__main__:
    print(squares)
```
Python list to dict transform:
```python
pairs = {i: i * i for i in range(3)}
print(pairs)
```
Rust map via iterator:
```rust
use std::collections::HashMap;

fn main() {
    let pairs: HashMap<i32, i32> = (0..3).map(|i| (i, i * i)).collect();
    println!("{:?}", pairs);
}
```
KCL mapping using schema defaults:
```kcl
schema Square:
    n: int
    sq: int = n * n

items: [{sq: int}] = [Square{n:0}, Square{n:1}, Square{n:2}]

__main__:
    print(items)
```

## Code warmup: conditionals and patterning
Python branching:
```python
def classify(x: int) -> str:
    if x % 2 == 0:
        return "even"
    return "odd"

print([classify(n) for n in range(4)])
```
Rust match as branching:
```rust
fn classify(x: i32) -> &'static str {
    match x % 2 {
        0 => "even",
        _ => "odd",
    }
}

fn main() {
    let labels: Vec<&str> = (0..4).map(classify).collect();
    println!("{:?}", labels);
}
```
KCL conditional:
```kcl
schema Parity:
    n: int
    label: str = "even" if n % 2 == 0 else "odd"

labels = [Parity{n:n}.label for n in [0,1,2,3]]

__main__:
    print(labels)
```
Python pattern matching (3.10+):
```python
def label(value):
    match value:
        case 0:
            return "zero"
        case 1 | 2:
            return "small"
        case _:
            return "other"

print(label(2))
```
Rust pattern match with guards:
```rust
fn label(value: i32) -> &'static str {
    match value {
        0 => "zero",
        1 | 2 => "small",
        n if n > 10 => "big",
        _ => "other",
    }
}

fn main() {
    println!("{}", label(12));
}
```
KCL match-like via if ladder:
```kcl
schema Label:
    value: int
    out: str = if value == 0 {"zero"} elif value in [1,2] {"small"} elif value > 10 {"big"} else {"other"}

__main__:
    print(Label{value:12}.out)
```

## Code warmup: small data structures
Python tuples and dataclasses:
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(2, 3)
print(p)
```
Rust struct:
```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 2, y: 3 };
    println!("{:?}", p);
}
```
KCL schema as data struct:
```kcl
schema Point:
    x: int
    y: int

p: Point = {x:2, y:3}

__main__:
    print(p)
```
Python dict-of-structs:
```python
points = {name: Point(idx, idx + 1) for idx, name in enumerate(["a","b"])}
print(points)
```
Rust HashMap of structs:
```rust
use std::collections::HashMap;

fn main() {
    let mut points: HashMap<&str, Point> = HashMap::new();
    points.insert("a", Point { x: 0, y: 1 });
    points.insert("b", Point { x: 1, y: 2 });
    println!("{:?}", points);
}
```
KCL mapping of schemas:
```kcl
ps = {name: Point{x:idx, y:idx+1} for idx, name in enumerate(["a","b"])}

__main__:
    print(ps)
```

## Notebook goal preview
- Objective: run KCL code cells inside Jupyter.
- Path: understand KCL runtime API, design kernel message loop (ZeroMQ), expose eval function, package as python module (pyo3 or wasmtime binding).
- Every later part will add the missing pieces.

## Exercises
1. Run each code block in its native toolchain (Python, Rust, KCL). Note where outputs differ.
2. Change the types (e.g., move from integers to strings) and check compiler or runtime errors. Log them.
3. Add a new field to the `Point` examples and see how each language handles defaults.
4. Sketch the kernel message flow you expect to build; keep it and revise in Part 09.


## Extra code reps (minimum viable muscle memory)

Python (4 quick blocks):
```python
# 1) Immutable vs mutable references (Python style)
nums = [1, 2, 3]
alias = nums
alias.append(4)
print(nums)

# 2) Simple function with type hints
from typing import List

def scale(xs: List[int], factor: int) -> List[int]:
    return [x * factor for x in xs]

print(scale(nums, 3))

# 3) Context manager for file IO
with open("/tmp/kcl_note.txt", "w") as f:
    f.write("hello kcl")

# 4) Dict comprehension for name mapping
people = ["ada", "grace", "linus", "yukihiro"]
lookup = {name: len(name) for name in people}
print(lookup)
```

Rust (4 blocks to mirror thinking):
```rust
// 1) Mutability and borrowing (no cloning yet)
fn mutate() {
    let mut nums = vec![1, 2, 3];
    nums.push(4);
    println!("{:?}", nums);
}
```
```rust
// 2) Function with generics and trait bounds
fn scale(xs: &[i32], factor: i32) -> Vec<i32> {
    xs.iter().map(|x| x * factor).collect()
}
```
```rust
// 3) Basic file IO with ?
use std::fs::File;
use std::io::Write;

fn write_note() -> std::io::Result<()> {
    let mut file = File::create("/tmp/kcl_note.txt")?;
    file.write_all(b"hello kcl")?;
    Ok(())
}
```
```rust
// 4) HashMap comprehension equivalent
use std::collections::HashMap;

fn name_lengths(names: &[&str]) -> HashMap<&str, usize> {
    names.iter().map(|n| (*n, n.len())).collect()
}
```

KCL (4 blocks to tie it back):
```kcl
# 1) Mutable-like update via let rebinding
nums = [1, 2, 3]
nums = nums + [4]
print(nums)
```
```kcl
# 2) Function-like schema (KCL functions are expressions)
scale = lambda xs, factor: [x * factor for x in xs]
print(scale([1,2,3], 3))
```
```kcl
# 3) File IO via std.fs (if enabled in your runtime env)
# fs = import("std/fs.k")
# fs.write("/tmp/kcl_note.txt", "hello kcl")
```
```kcl
# 4) Dict comprehension mirrors Python
people = ["ada", "grace", "linus", "yukihiro"]
lookup = {name: len(name) for name in people}
print(lookup)
```
