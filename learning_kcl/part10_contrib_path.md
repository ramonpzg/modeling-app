# Part 10 — Contribution path and kata library

This last part turns the lessons into actionable steps. Each section includes code snippets to keep the rhythm.

## Repo hygiene
Python script to run formatter/tests:
```python
import subprocess
subprocess.run(["npm","test"])
subprocess.run(["cargo","test"], cwd="rust/kcl-lib")
```
Rust using `just` tasks (conceptual):
```rust
fn main() {
    // run with: just fmt && just test
}
```
KCL config describing tasks:
```kcl
tasks: [str] = ["npm test", "cargo test"]

__main__:
    print(tasks)
```
Python lint command example:
```python
subprocess.run(["python","-m","ruff","."])
```
Rust fmt and clippy:
```rust
fn main() {
    // run: cargo fmt && cargo clippy
}
```
KCL reminder for formatting policy:
```kcl
format_policy: str = "cargo fmt, npm lint"
```

## Reading the codebase
Python quick grep:
```python
import pathlib
matches = [p for p in pathlib.Path("rust/kcl-lib").rglob("*lexer*")]
print(matches[:5])
```
Rust search using `rg` (command string):
```rust
fn main() {
    // run: rg "Expr" rust/kcl-lib/crates
}
```
KCL list of directories to study:
```kcl
study_paths: [str] = ["crates/kcl-language/src/lexer", "crates/kcl-language/src/parser", "crates/kclvm"]
```
Python reading a file:
```python
path = pathlib.Path("rust/kcl-lib/README.md")
print(path.read_text().splitlines()[:3])
```
Rust reading file:
```rust
use std::fs;
fn main() { let data = fs::read_to_string("rust/kcl-lib/README.md").unwrap(); println!("{}", &data[..50]); }
```
KCL storing docs summary:
```kcl
docs_intro: str = "kcl-lib README overview"
```

## Katas for Rust
Python unit test harness for translation:
```python
def add(a:int,b:int)->int: return a+b
assert add(1,2)==3
```
Rust equivalent kata:
```rust
#[cfg(test)]
mod tests {
    #[test]
    fn add() { assert_eq!(1+2, 3); }
}
```
KCL assertion:
```kcl
check:
    assert 1 + 2 == 3
```
Python iterator kata:
```python
nums = [n*n for n in range(5)]
print(nums)
```
Rust iterator kata:
```rust
fn main() { let nums: Vec<i32> = (0..5).map(|n| n*n).collect(); println!("{:?}", nums); }
```
KCL iterator kata:
```kcl
nums = [n*n for n in range(5)]
```
Python trait/protocol kata:
```python
from typing import Protocol
class Describable(Protocol):
    def describe(self) -> str: ...
class Item: 
    def describe(self)->str: 
        return "item"
def show(x: Describable): 
    print(x.describe())
show(Item())
```
Rust trait kata:
```rust
trait Describable { fn describe(&self) -> String; }
struct Item;
impl Describable for Item { fn describe(&self) -> String { "item".into() } }
fn main() { println!("{}", Item.describe()); }
```
KCL describe field:
```kcl
schema Item:
    describe: str = "item"

__main__:
    print(Item{}.describe)
```

## Katas for KCL
Python config style vs KCL:
```python
config = {"db":{"host":"localhost","port":5432}}
```
Rust struct for config:
```rust
#[derive(Debug)]
struct Db { host: String, port: u16 }
```
KCL schema config:
```kcl
schema Db:
    host: str = "localhost"
    port: int = 5432

db: Db = {}
```
Python overlay config:
```python
prod = config | {"db":{"host":"prod"}}
```
Rust builder pattern:
```rust
struct DbBuilder { host: String, port: u16 }
impl DbBuilder { fn new() -> Self { Self { host:"localhost".into(), port:5432 } } fn host(mut self, h:&str)->Self{ self.host=h.into(); self } }
```
KCL overlay:
```kcl
prod_db: Db = {host:"prod"}
```
Python validation kata:
```python
assert prod["db"]["port"] > 0
```
Rust validation kata:
```rust
fn validate(db: &Db) { assert!(db.port > 0); }
```
KCL check:
```kcl
check:
    assert prod_db.port > 0
```

## Contribution steps
Python script to open issue via CLI:
```python
import webbrowser
webbrowser.open("https://github.com/zoo-project/modeling-app/issues/new")
```
Rust note on git workflow:
```rust
fn main() {
    // branch, commit, push; run cargo fmt/test before PR
}
```
KCL contribution checklist as data:
```kcl
steps: [str] = [
    "Create branch",
    "Write tests",
    "Run cargo fmt/clippy",
    "Run npm/vitest if touching frontend",
    "Document changes",
]
```
Python example of describing change:
```python
description = "Add KCL kernel prototype"
print(description)
```
Rust commit message guideline:
```rust
fn main() {
    // prefer imperative: "Add kernel request handler"
}
```
KCL note:
```kcl
commit_style: str = "imperative mood"
```

## Exercises
1. Turn the katas into scripts you can run automatically after each save (Python: pytest; Rust: cargo test; KCL: kcl run).
2. Open a draft PR adding docs or tests; treat this file as your checklist.
3. Implement one small Rust change in `kcl-lib` (e.g., new unit test) and mirror its behavior in KCL.
4. Draft the first notebook kernel issue using the templates above and paste the exact steps you will take.


## Extra code reps (contribution muscle memory)

Python (4 blocks):
```python
# 1) argparse skeleton
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--flag", action="store_true")
args = parser.parse_args([])
print(args)
```
```python
# 2) packaging stub (setup.cfg style)
setup_cfg = """
[metadata]
name = kcl_kernel
version = 0.0.1
"""
print(setup_cfg.strip())
```
```python
# 3) logging baseline
import logging
logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)
log.info("hello")
```
```python
# 4) simple benchmark
import timeit
print(timeit.timeit('sum(range(1000))', number=100))
```

Rust (4 blocks):
```rust
// 1) clap CLI skeleton
use clap::Parser;
#[derive(Parser, Debug)]
struct Args { #[arg(long)] flag: bool }
```
```rust
// 2) cargo manifest snippet
// [package]
// name = "kcl-kernel"
// version = "0.0.1"
```
```rust
// 3) tracing
use tracing::info;
fn log_line() { info!("hello"); }
```
```rust
// 4) criterion benchmark
// use criterion::{criterion_group, criterion_main, Criterion};
```

KCL (4 blocks):
```kcl
# 1) CLI config map
cli = {flag: true}
print(cli)
```
```kcl
# 2) Package metadata placeholder
meta = {name: "kcl-kernel", version: "0.0.1"}
print(meta)
```
```kcl
# 3) Logging via print
log = lambda msg: print("INFO", msg)
log("hello")
```
```kcl
# 4) Simple perf check
start = 0
# imagine timing comprehension
nums = [n for n in range(1000)]
print(len(nums))
```
