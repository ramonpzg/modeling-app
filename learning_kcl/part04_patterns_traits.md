# Part 04 — Patterns, traits, and code reuse tri-language edition

Traits give Rust its expressive power; protocols and ABCs do the job in Python; KCL leans on schemas and attribute presence. This part keeps examples dense: at least four snippets per language.

## Pattern matching
Python structural pattern matching:
```python
def classify(item):
    match item:
        case {"kind": "point", "x": x, "y": y}:
            return x + y
        case [first, *_]:
            return first
        case _:
            return 0

print(classify({"kind":"point","x":1,"y":2}))
```
Rust `match` with destructuring:
```rust
enum Shape {
    Point { x: i32, y: i32 },
    List(Vec<i32>),
    None,
}

fn classify(shape: Shape) -> i32 {
    match shape {
        Shape::Point { x, y } => x + y,
        Shape::List(v) if !v.is_empty() => v[0],
        _ => 0,
    }
}

fn main() {
    println!("{}", classify(Shape::Point { x: 1, y: 2 }));
}
```
KCL conditional destructuring via schema checks:
```kcl
schema Point:
    x: int
    y: int

schema Shape:
    point?: Point
    list?: [int]

shape1: Shape = {point: Point{x:1, y:2}}
value = shape1.point.x + shape1.point.y if shape1.point else (shape1.list[0] if shape1.list else 0)

__main__:
    print(value)
```
Python match with guards:
```python
def describe(n):
    match n:
        case x if x > 10:
            return "large"
        case 0:
            return "zero"
        case _:
            return "small"

print(describe(12))
```
Rust match guards:
```rust
fn describe(n: i32) -> &'static str {
    match n {
        x if x > 10 => "large",
        0 => "zero",
        _ => "small",
    }
}

fn main() {
    println!("{}", describe(12));
}
```
KCL guard-ish conditional:
```kcl
schema Describe:
    n: int
    label: str = "large" if n > 10 else ("zero" if n == 0 else "small")

__main__:
    print(Describe{n:12}.label)
```

## Traits, protocols, and schemas
Python protocols:
```python
from typing import Protocol

class Printable(Protocol):
    def render(self) -> str: ...

class User:
    def __init__(self, name: str):
        self.name = name
    def render(self) -> str:
        return f"User:{self.name}"

def show(item: Printable) -> None:
    print(item.render())

show(User("alice"))
```
Rust traits:
```rust
trait Render {
    fn render(&self) -> String;
}

struct User { name: String }
impl Render for User {
    fn render(&self) -> String { format!("User:{}", self.name) }
}

fn show(item: &impl Render) {
    println!("{}", item.render());
}

fn main() {
    show(&User { name: "alice".into() });
}
```
KCL schema with computed method-like field:
```kcl
schema User:
    name: str
    render: str = "User:${name}"

schema Show:
    item: User

example: Show = {item: User{name:"alice"}}

__main__:
    print(example.item.render)
```
Python ABC alternative:
```python
from abc import ABC, abstractmethod

class Service(ABC):
    @abstractmethod
    def handle(self, msg: str) -> str: ...

class Echo(Service):
    def handle(self, msg: str) -> str:
        return msg.upper()

def dispatch(svc: Service, msg: str) -> str:
    return svc.handle(msg)

print(dispatch(Echo(), "ping"))
```
Rust trait objects:
```rust
trait Service {
    fn handle(&self, msg: &str) -> String;
}

struct Echo;
impl Service for Echo {
    fn handle(&self, msg: &str) -> String { msg.to_uppercase() }
}

fn dispatch(svc: &dyn Service, msg: &str) -> String {
    svc.handle(msg)
}

fn main() {
    println!("{}", dispatch(&Echo, "ping"));
}
```
KCL dispatch via type field:
```kcl
schema Service:
    kind: str
    msg: str
    out: str = msg.upper() if kind == "echo" else msg

svc: Service = {kind:"echo", msg:"ping"}

__main__:
    print(svc.out)
```

## Generics and reuse
Python generics via typing only:
```python
from typing import TypeVar, Iterable, List
T = TypeVar('T')

def repeat(item: T, n: int) -> List[T]:
    return [item for _ in range(n)]

print(repeat("x", 3))
```
Rust generics enforced at compile time:
```rust
fn repeat<T: Clone>(item: T, n: usize) -> Vec<T> {
    (0..n).map(|_| item.clone()).collect()
}

fn main() {
    println!("{:?}", repeat("x", 3));
}
```
KCL reuse via parameterized schema:
```kcl
schema RepeatStr:
    item: str
    n: int
    out: [str] = [item for _ in range(n)]

example: RepeatStr = {item:"x", n:3}

__main__:
    print(example.out)
```
Python duck-typed sorting function:
```python
def max_len(strings):
    return max(strings, key=len)

print(max_len(["a","abcd","zz"]))
```
Rust generic with trait bound:
```rust
fn max_len<'a>(strings: &'a [String]) -> Option<&'a String> {
    strings.iter().max_by_key(|s| s.len())
}

fn main() {
    let data = vec!["a".to_string(), "abcd".into(), "zz".into()];
    println!("{:?}", max_len(&data));
}
```
KCL selecting by length:
```kcl
items: [str] = ["a", "abcd", "zz"]
max_item = max(items, key=lambda s: len(s))

__main__:
    print(max_item)
```

## Iterators and adapters
Python generators:
```python
def squares(n):
    for i in range(n):
        yield i * i

print(list(squares(4)))
```
Rust iterators with adapters:
```rust
fn main() {
    let res: Vec<i32> = (0..4).map(|i| i * i).filter(|v| v % 2 == 0).collect();
    println!("{:?}", res);
}
```
KCL comprehensions:
```kcl
res = [i*i for i in range(4) if (i*i) % 2 == 0]

__main__:
    print(res)
```
Python itertools chain:
```python
import itertools as it
vals = list(it.chain([1,2], [3,4]))
print(vals)
```
Rust chaining iterators:
```rust
fn main() {
    let vals: Vec<i32> = [1,2].iter().chain([3,4].iter()).cloned().collect();
    println!("{:?}", vals);
}
```
KCL concatenation:
```kcl
vals = [1,2] + [3,4]

__main__:
    print(vals)
```

## Exercises
1. Add a new trait/protocol for serialization in all three languages and implement for two structs/schemas.
2. Convert the iterator examples to process structs instead of ints; in Rust, implement `IntoIterator` for a custom type.
3. Extend the pattern matching sections with an error variant and ensure exhaustive handling in Rust.
4. Relate each pattern to how you would handle AST nodes in `rust/kcl-lib` (e.g., `Expr` enums and visitor traits).


## Extra code reps (traits, enums, and pattern shape)

Python (4 blocks):
```python
# 1) Protocol-style duck typing
from typing import Protocol

class Greeter(Protocol):
    def greet(self) -> str:
        ...

class ConsoleGreeter:
    def greet(self) -> str:
        return "hi"

def say(g: Greeter):
    print(g.greet())

say(ConsoleGreeter())
```
```python
# 2) Enum via Enum class
from enum import Enum

class Direction(Enum):
    NORTH = "N"
    SOUTH = "S"

print(Direction.NORTH.value)
```
```python
# 3) Pattern matching enums
match Direction.NORTH:
    case Direction.NORTH:
        print("north")
    case _:
        print("other")
```
```python
# 4) Dataclass with method
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int
    def norm(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

print(Point(3, 4).norm())
```

Rust (4 blocks):
```rust
// 1) Trait and impl
trait Greeter {
    fn greet(&self) -> String;
}

struct Console;

impl Greeter for Console {
    fn greet(&self) -> String { "hi".into() }
}

fn say<G: Greeter>(g: &G) {
    println!("{}", g.greet());
}
```
```rust
// 2) Enum with data
#[derive(Debug)]
enum Direction {
    North,
    South,
}

fn print_dir(dir: Direction) {
    match dir {
        Direction::North => println!("north"),
        Direction::South => println!("south"),
    }
}
```
```rust
// 3) Pattern matching tuples
fn classify(pair: (i32, i32)) {
    match pair {
        (0, 0) => println!("origin"),
        (x, y) if x == y => println!("diagonal"),
        _ => println!("somewhere"),
    }
}
```
```rust
// 4) Struct with method
struct Point { x: i32, y: i32 }
impl Point {
    fn norm(&self) -> f64 {
        ((self.x.pow(2) + self.y.pow(2)) as f64).sqrt()
    }
}
```

KCL (4 blocks):
```kcl
# 1) Behavior via schema methods
schema Greeter:
    greet: func() -> str

schema Console:
    greet = lambda : "hi"

say = lambda g: print(g.greet())
say(Console{})
```
```kcl
# 2) Enum-like via union and constants
Direction = "N" | "S"
print("N" in Direction)
```
```kcl
# 3) Pattern matching tuples
pair = (1, 1)
match pair:
    case (0,0):
        print("origin")
    case (x, y) if x == y:
        print("diagonal")
    case _:
        print("somewhere")
```
```kcl
# 4) Schema with method
schema Point:
    x: int
    y: int
    norm = lambda self: ((self.x ** 2 + self.y ** 2) ** 0.5)

print(Point{x:3, y:4}.norm())
```
