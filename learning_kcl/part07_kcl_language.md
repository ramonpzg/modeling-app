# Part 07 — KCL language tour with Rust and Python mirrors

Time to focus on KCL syntax and semantics while keeping Rust and Python analogies.

## Types and schemas
Python dataclass:
```python
from dataclasses import dataclass
@dataclass
class Server:
    host: str
    port: int

print(Server("localhost", 8080))
```
Rust struct:
```rust
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
}

fn main() {
    let s = Server { host: "localhost".into(), port: 8080 };
    println!("{:?}", s);
}
```
KCL schema with defaults and annotations:
```kcl
schema Server:
    host: str = "localhost"
    port: int = 8080

srv: Server = {}

__main__:
    print(srv)
```
Python nested dataclass:
```python
@dataclass
class TLS:
    enabled: bool = False
    cert: str | None = None

@dataclass
class Config:
    server: Server
    tls: TLS

print(Config(Server("localhost",8080), TLS()))
```
Rust nested struct:
```rust
#[derive(Debug)]
struct Tls { enabled: bool, cert: Option<String> }
#[derive(Debug)]
struct Config { server: Server, tls: Tls }

fn main() {
    let cfg = Config { server: Server { host: "localhost".into(), port: 8080 }, tls: Tls { enabled: false, cert: None } };
    println!("{:?}", cfg);
}
```
KCL nested schemas:
```kcl
schema Tls:
    enabled: bool = False
    cert?: str

schema Config:
    server: Server
    tls: Tls = Tls{}

cfg: Config = {server: Server{}, tls: Tls{enabled:True, cert:"path"}}

__main__:
    print(cfg)
```

## Expressions and operators
Python arithmetic and bools:
```python
print((1 + 2) * 3)
print(True and False or True)
```
Rust equivalents:
```rust
fn main() {
    println!("{}", (1 + 2) * 3);
    println!("{}", (true && false) || true);
}
```
KCL expressions:
```kcl
value: int = (1 + 2) * 3
flag: bool = (True and False) or True

__main__:
    print([value, flag])
```
Python string formatting:
```python
name = "kcl"
print(f"hello {name}")
```
Rust formatting:
```rust
fn main() {
    let name = "kcl";
    println!("hello {}", name);
}
```
KCL interpolation:
```kcl
name: str = "kcl"
msg: str = "hello ${name}"

__main__:
    print(msg)
```
Python list/set/dict expressions:
```python
lst = [x*x for x in range(3)]
set_vals = {x for x in lst}
dct = {x: x*x for x in range(3)}
print(lst, set_vals, dct)
```
Rust collections:
```rust
use std::collections::{HashSet, HashMap};
fn main() {
    let lst: Vec<i32> = (0..3).map(|x| x*x).collect();
    let set: HashSet<i32> = lst.iter().cloned().collect();
    let map: HashMap<i32,i32> = (0..3).map(|x| (x, x*x)).collect();
    println!("{:?} {:?} {:?}", lst, set, map);
}
```
KCL collections:
```kcl
lst = [x*x for x in range(3)]
set_vals = {x: None for x in lst}.keys()
dct = {x: x*x for x in range(3)}

__main__:
    print([lst, list(set_vals), dct])
```

## Control flow
Python conditional and loop:
```python
for i in range(3):
    if i % 2 == 0:
        print("even")
    else:
        print("odd")
```
Rust control flow:
```rust
fn main() {
    for i in 0..3 {
        if i % 2 == 0 {
            println!("even");
        } else {
            println!("odd");
        }
    }
}
```
KCL loops are expression-based:
```kcl
labels = ["even" if i % 2 == 0 else "odd" for i in range(3)]

__main__:
    print(labels)
```
Python match for enums:
```python
from enum import Enum
class State(Enum):
    START = 1
    STOP = 2

def handle(state: State):
    match state:
        case State.START:
            return "go"
        case State.STOP:
            return "halt"

print(handle(State.START))
```
Rust enum match:
```rust
enum State { Start, Stop }

fn handle(state: State) -> &'static str {
    match state {
        State::Start => "go",
        State::Stop => "halt",
    }
}

fn main() {
    println!("{}", handle(State::Start));
}
```
KCL enum via choices:
```kcl
schema State:
    kind: str

fn handle(kind: str) -> str:
    if kind == "start" {"go"} elif kind == "stop" {"halt"} else {"unknown"}

__main__:
    print(handle("start"))
```

## Modules and imports in KCL
Python package usage already known.
Rust `mod` was shown earlier.
KCL imports with packages:
```kcl
import "./server.k"

schema App:
    server: Server

app: App = {server: Server{host:"prod", port:80}}

__main__:
    print(app.server.host)
```
Python relative imports and aliasing:
```python
from mypkg.server import Server as PyServer
print(PyServer("prod", 80))
```
Rust path imports:
```rust
use crate::server::Server as RsServer;
fn main() { println!("{:?}", RsServer { host: "prod".into(), port: 80 }); }
```
KCL aliasing via import rename:
```kcl
import "./server.k" as srv

__main__:
    print(srv.Server{host:"prod", port:80}.host)
```

## Validation and policy
Python runtime checks:
```python
def validate(cfg: Server):
    assert cfg.port > 0

validate(Server("localhost",8080))
```
Rust compile/runtime check:
```rust
fn validate(cfg: &Server) {
    assert!(cfg.port > 0);
}

fn main() {
    let s = Server { host: "localhost".into(), port: 8080 };
    validate(&s);
}
```
KCL `check` block:
```kcl
srv: Server = {}

check:
    assert srv.port > 0
```
Python type coercion error example:
```python
try:
    Server("host", "oops")
except TypeError as e:
    print(e)
```
Rust type mismatch compile-time:
```rust
// let bad = Server { host: "host".into(), port: "oops" }; // won't compile
```
KCL type mismatch caught by checker:
```kcl
# bad: Server = {host:"host", port:"oops"}
```

## Exercises
1. Rewrite a `rust/kcl-lib` AST node struct in KCL schema form and compare readability.
2. Extend the Server schema with optional TLS and logging fields; mirror in Rust and Python.
3. Create a KCL file that imports another and overrides defaults; note overlay semantics.
4. Add checks to ensure ports are within range across all languages.


## Extra code reps (language basics side-by-side)

Python (4 blocks):
```python
# 1) dict + list comprehension combo
people = [{"name": n, "age": i} for i, n in enumerate(["ada", "grace"])]
print(people)
```
```python
# 2) function default args
def greet(name: str, loud: bool = False) -> str:
    return name.upper() if loud else name

print(greet("kcl", loud=True))
```
```python
# 3) mini class with __repr__
class User:
    def __init__(self, name: str):
        self.name = name
    def __repr__(self):
        return f"User({self.name})"

print(User("ada"))
```
```python
# 4) list filtering
nums = [n for n in range(10) if n % 3 == 0]
print(nums)
```

Rust (4 blocks):
```rust
// 1) Vec of structs with iterator map
#[derive(Debug)]
struct Person { name: &'static str, age: usize }
fn people() {
    let ps: Vec<Person> = ["ada", "grace"].iter().enumerate().map(|(i, n)| Person { name: n, age: i }).collect();
    println!("{:?}", ps);
}
```
```rust
// 2) Function with default via Option
fn greet(name: &str, loud: Option<bool>) -> String {
    match loud.unwrap_or(false) {
        true => name.to_uppercase(),
        false => name.into(),
    }
}
```
```rust
// 3) Struct with Display
use std::fmt;
struct User { name: String }
impl fmt::Display for User {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "User({})", self.name)
    }
}
```
```rust
// 4) Iterator filter
fn multiples_of_three() {
    let nums: Vec<i32> = (0..10).filter(|n| n % 3 == 0).collect();
    println!("{:?}", nums);
}
```

KCL (4 blocks):
```kcl
# 1) dict + list comprehension
people = [{name: n, age: idx} for idx, n in enumerate(["ada", "grace"])]
print(people)
```
```kcl
# 2) function default
Greet = lambda name, loud=false: name.upper() if loud else name
print(Greet("kcl", loud=true))
```
```kcl
# 3) schema display
schema User:
    name: str

print(User{name="ada"})
```
```kcl
# 4) list filtering
nums = [n for n in range(10) if n % 3 == 0]
print(nums)
```
