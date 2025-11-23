# Part 02 — Rust crash and KCL echo with Python goggles

You learn languages faster by triangulation. Each section shows the Rust move, the Python anchor, and how KCL expresses similar intent. Four code reps per language keep your fingers honest.

## Binding and mutability
Python allows rebinding by default:
```python
value = 1
value = 2
value += 3
print(value)
```
Rust separates immutability and mutability:
```rust
fn main() {
    let x = 1;
    let mut y = 2;
    // x = 3; // compile error: cannot assign to immutable
    y += 3;
    println!("{}", y);
}
```
KCL defaults to immutability but supports override in schemas:
```kcl
schema Config:
    x: int = 1
    y: int = 2

c: Config = {x:1, y:2}
# x is fixed; you set via overrides at instantiation time
c2: Config = {x:1, y:5}

__main__:
    print(c2.y)
```
Python explicit copy to mimic immutability:
```python
from copy import deepcopy
orig = {"x": 1, "y": 2}
new = deepcopy(orig)
new["y"] = 5
print(orig, new)
```
Rust shadowing versus mutation:
```rust
fn main() {
    let score = 10;
    let score = score + 5; // shadowing creates new binding
    let mut mutable = 3;
    mutable = mutable * 2;
    println!("{} {}", score, mutable);
}
```
KCL overlay behavior:
```kcl
schema Score:
    base: int
    bonus: int = 5
    total: int = base + bonus

player1: Score = {base:10}
player2: Score = {base:10, bonus:10}

__main__:
    print([player1.total, player2.total])
```

## Types and inference
Python typing hints are optional:
```python
from typing import List
nums: List[int] = [1, 2, 3]
result = [n * 2 for n in nums]
print(result)
```
Rust has explicit types with inference when clear:
```rust
fn main() {
    let nums = vec![1, 2, 3];
    let doubled: Vec<i32> = nums.iter().map(|n| n * 2).collect();
    println!("{:?}", doubled);
}
```
KCL has static types baked into schemas:
```kcl
schema Numbers:
    items: [int]
    doubled: [int] = [n * 2 for n in items]

example: Numbers = {items:[1,2,3]}

__main__:
    print(example.doubled)
```
Python runtime type error example:
```python
def half(x: int) -> float:
    return x / 2

print(half(3))
print(half("oops"))  # runtime error
```
Rust compile-time catch:
```rust
fn half(x: i32) -> f64 {
    (x as f64) / 2.0
}

fn main() {
    println!("{}", half(3));
    // println!("{}", half("oops")); // will not compile
}
```
KCL validation at configuration time:
```kcl
schema Half:
    x: int
    val: float = x / 2

valid: Half = {x:4}
# invalid: Half = {x:"oops"}  # type checker will stop you

__main__:
    print(valid.val)
```

## Functions and signatures
Python flexible defaults:
```python
def greet(name: str = "world", excited: bool = False) -> str:
    suffix = "!" if excited else "."
    return f"hi {name}{suffix}"

print(greet())
print(greet("kcl", True))
```
Rust functions with clear lifetimes (not yet needed here):
```rust
fn greet(name: &str, excited: bool) -> String {
    let suffix = if excited {"!"} else {"."};
    format!("hi {}{}", name, suffix)
}

fn main() {
    println!("{}", greet("world", false));
    println!("{}", greet("kcl", true));
}
```
KCL functions via schemas or top-level defs:
```kcl
schema Greeter:
    name: str = "world"
    excited: bool = False
    msg: str = "hi ${name}${"!" if excited else "."}"

__main__:
    print(Greeter{}.msg)
    print(Greeter{name:"kcl", excited:True}.msg)
```
Python higher-order function sample:
```python
def apply_twice(func, x):
    return func(func(x))

print(apply_twice(lambda v: v + 1, 3))
```
Rust higher-order via closures:
```rust
fn apply_twice<F>(func: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    func(func(x))
}

fn main() {
    println!("{}", apply_twice(|v| v + 1, 3));
}
```
KCL composition using expressions:
```kcl
schema Increment:
    x: int
    val: int = (x + 1) + 1

__main__:
    print(Increment{x:3}.val)
```

## Modules and visibility
Python module import:
```python
# file: utils.py
def ping() -> str:
    return "pong"

# file: main.py
from utils import ping
print(ping())
```
Rust modules with `mod` and `pub`:
```rust
// utils.rs
pub fn ping() -> &'static str { "pong" }

// main.rs
mod utils;
fn main() {
    println!("{}", utils::ping());
}
```
KCL package-like structuring:
```kcl
# utils.k
schema Utils:
    ping: str = "pong"

# main.k
import "./utils.k"

__main__:
    print(Utils{}.ping)
```
Python package with `__init__.py` exposing symbols:
```python
# __init__.py
from .utils import ping

# consumer.py
from mypkg import ping
print(ping())
```
Rust visibility control example:
```rust
mod secrets {
    pub fn show() -> &'static str { "public" }
    fn hidden() -> &'static str { "private" }
}

fn main() {
    println!("{}", secrets::show());
    // secrets::hidden(); // compile error: private function
}
```
KCL import with selective exposure:
```kcl
# secrets.k
schema Public:
    info: str = "public"

schema Hidden:
    info: str = "private"

# main.k
import "./secrets.k"

__main__:
    print(Public{}.info)
```

## Exercises
1. Recreate each code block in your editor, run it, and log any warnings or compiler errors.
2. Add type annotations to the Python snippets and mirror them in Rust and KCL.
3. Change visibility (public/private) in Rust and KCL examples, observe compiler feedback.
4. Translate a Python higher-order function that captures state (closure over variable) into Rust with `move` and into KCL with schema defaults.


## Extra code reps (control flow and binding)

Python (4 blocks):
```python
# 1) if/elif/else quick check
val = 7
if val % 2 == 0:
    print("even")
elif val % 3 == 0:
    print("divisible by 3")
else:
    print("odd")
```
```python
# 2) for with enumerate
names = ["kcl", "rust", "python"]
for idx, name in enumerate(names):
    print(idx, name)
```
```python
# 3) while loop with break/continue
n = 0
while True:
    n += 1
    if n % 2 == 0:
        continue
    if n > 5:
        break
    print(n)
```
```python
# 4) match via structural pattern (3.10+)
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def classify(p: Point):
    match p:
        case Point(x=0, y=0):
            return "origin"
        case Point(x, y) if x == y:
            return "diagonal"
        case _:
            return "somewhere"

print(classify(Point(1, 1)))
```

Rust (4 blocks):
```rust
// 1) if/else
fn parity(val: i32) {
    if val % 2 == 0 {
        println!("even");
    } else if val % 3 == 0 {
        println!("divisible by 3");
    } else {
        println!("odd");
    }
}
```
```rust
// 2) for with enumerate
fn loop_names() {
    for (idx, name) in ["kcl", "rust", "python"].iter().enumerate() {
        println!("{} {}", idx, name);
    }
}
```
```rust
// 3) while and loop
fn count_odds() {
    let mut n = 0;
    loop {
        n += 1;
        if n % 2 == 0 {
            continue;
        }
        if n > 5 {
            break;
        }
        println!("{}", n);
    }
}
```
```rust
// 4) match with guards
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn classify(p: Point) -> &'static str {
    match p {
        Point { x: 0, y: 0 } => "origin",
        Point { x, y } if x == y => "diagonal",
        _ => "somewhere",
    }
}
```

KCL (4 blocks):
```kcl
# 1) if/elif/else
val = 7
if val % 2 == 0:
    print("even")
elif val % 3 == 0:
    print("divisible by 3")
else:
    print("odd")
```
```kcl
# 2) for with enumerate
names = ["kcl", "rust", "python"]
for idx, name in enumerate(names):
    print(idx, name)
```
```kcl
# 3) while-like using loop and break
n = 0
while true:
    n = n + 1
    if n % 2 == 0:
        continue
    if n > 5:
        break
    print(n)
```
```kcl
# 4) match via pattern match
schema Point:
    x: int
    y: int

p = Point{x:1, y:1}
match p:
    case {x:0, y:0}:
        print("origin")
    case {x, y} if x == y:
        print("diagonal")
    case _:
        print("somewhere")
```
