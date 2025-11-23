# Part 08 — kcl-lib architecture with cross-language anchors

We zoom into the repo to connect language concepts to code. Use these examples to navigate modules.

## Where things live
- Lexer/Parser: `rust/kcl-lib/crates/kcl-language/src/lexer`, `parser`.
- AST and semantic analysis: `rust/kcl-lib/crates/kcl-language/src/ast` and `resolver`.
- VM/evaluator: `rust/kcl-lib/crates/kclvm`.
- WASM bindings: `packages/kcl-wasm`.

## Tokenizing
Python quick tokenizer for comparison:
```python
import re
pattern = re.compile(r"\w+|\+|\-")
print(pattern.findall("a + b - c"))
```
Rust lexer usage (conceptual simplified):
```rust
use kcl_language::lexer::Lexer;
fn main() {
    let mut lexer = Lexer::new("a + b - c");
    while let Some(tok) = lexer.next_token() {
        println!("{:?}", tok);
    }
}
```
KCL tokens visualized manually:
```kcl
tokens: [str] = ["IDENT(a)", "PLUS", "IDENT(b)", "MINUS", "IDENT(c)"]

__main__:
    print(tokens)
```
Python custom lexer class:
```python
class Lexer:
    def __init__(self, text):
        self.text = text.split()
    def tokens(self):
        for part in self.text:
            yield part

print(list(Lexer("a + b").tokens()))
```
Rust tokens via logos crate (analogy):
```rust
use logos::Logos;
#[derive(Logos, Debug, PartialEq)]
enum Token {
    #[regex(r"[a-zA-Z_]+")]
    Ident,
    #[token("+")]
    Plus,
    #[token("-")]
    Minus,
}

fn main() {
    for tok in Token::lexer("a + b") {
        println!("{:?}", tok);
    }
}
```
KCL config for keywords list:
```kcl
keywords: [str] = ["schema", "import", "check"]

__main__:
    print(keywords)
```

## AST building
Python AST representation:
```python
class Expr: ...
class Binary(Expr):
    def __init__(self, left, op, right):
        self.left, self.op, self.right = left, op, right

expr = Binary("a", "+", "b")
print(expr.op)
```
Rust AST struct:
```rust
#[derive(Debug)]
struct Binary<'a> {
    left: &'a str,
    op: char,
    right: &'a str,
}

fn main() {
    let expr = Binary { left: "a", op: '+', right: "b" };
    println!("{:?}", expr);
}
```
KCL AST-as-config idea:
```kcl
schema Binary:
    left: str
    op: str
    right: str

expr: Binary = {left:"a", op:"+", right:"b"}

__main__:
    print(expr.op)
```
Python visitor pattern:
```python
class Visitor:
    def visit_binary(self, bin):
        return f"{bin.left}{bin.op}{bin.right}"

print(Visitor().visit_binary(expr))
```
Rust visitor trait (simplified):
```rust
trait Visitor {
    fn visit_binary(&mut self, bin: &Binary) -> String;
}

struct Printer;
impl Visitor for Printer {
    fn visit_binary(&mut self, bin: &Binary) -> String {
        format!("{}{}{}", bin.left, bin.op, bin.right)
    }
}

fn main() {
    let mut p = Printer;
    println!("{}", p.visit_binary(&Binary { left:"a", op:'+', right:"b" }));
}
```
KCL printer via string interpolation:
```kcl
render: str = f"{expr.left}{expr.op}{expr.right}"

__main__:
    print(render)
```

## Evaluation pipeline
Python interpreter sketch:
```python
def eval_expr(bin: Binary) -> int:
    if bin.op == "+":
        return int(bin.left) + int(bin.right)
    raise ValueError("unknown op")

print(eval_expr(Binary("1","+","2")))
```
Rust evaluator:
```rust
fn eval_expr(bin: &Binary) -> i32 {
    match bin.op {
        '+' => bin.left.parse::<i32>().unwrap() + bin.right.parse::<i32>().unwrap(),
        _ => panic!("unknown op"),
    }
}
```
KCL evaluation as configuration:
```kcl
schema Add:
    a: int
    b: int
    result: int = a + b

add: Add = {a:1, b:2}

__main__:
    print(add.result)
```
Python error propagation in interpreter:
```python
def eval_expr_safe(bin: Binary) -> int:
    if bin.op not in {"+", "-"}:
        raise ValueError("bad op")
    return int(bin.left) + int(bin.right)
```
Rust Result return:
```rust
fn eval_expr_safe(bin: &Binary) -> Result<i32, String> {
    match bin.op {
        '+' => Ok(bin.left.parse::<i32>().unwrap() + bin.right.parse::<i32>().unwrap()),
        '-' => Ok(bin.left.parse::<i32>().unwrap() - bin.right.parse::<i32>().unwrap()),
        _ => Err("bad op".into()),
    }
}
```
KCL check block to fail fast:
```kcl
schema AddSafe:
    a: int
    b: int
    op: str
    result: int = a + b if op == "+" else a - b

check:
    assert op in ["+", "-"]
```

## Serialization and storage
Python JSON handling:
```python
import json
cfg = {"host":"localhost","port":8080}
print(json.dumps(cfg))
```
Rust serde:
```rust
use serde::{Serialize, Deserialize};
#[derive(Serialize, Deserialize, Debug)]
struct ServerCfg { host: String, port: u16 }
fn main() {
    let cfg = ServerCfg { host: "localhost".into(), port: 8080 };
    println!("{}", serde_json::to_string(&cfg).unwrap());
}
```
KCL to JSON via CLI:
```kcl
schema Server:
    host: str
    port: int

srv: Server = {host:"localhost", port:8080}

__main__:
    print(srv)
# run `kcl run config.k -o json`
```
Python YAML for configs:
```python
import yaml
print(yaml.safe_dump(cfg))
```
Rust yaml via serde_yaml:
```rust
fn main() {
    let cfg = ServerCfg { host:"localhost".into(), port:8080 };
    println!("{}", serde_yaml::to_string(&cfg).unwrap());
}
```
KCL YAML export conceptually similar via CLI flags.

## Exercises
1. Open `rust/kcl-lib/crates/kcl-language/src/ast` and map each struct to a KCL schema sketch to check understanding.
2. Trace a simple expression through lexer -> parser -> resolver -> evaluator in Rust; annotate which code blocks in this doc mirror each phase.
3. Serialize a KCL config to JSON and compare the shape to the Rust `serde` output.
4. Add a new token to the lexer (e.g., `**` for power) and update the Python and KCL examples to reflect it.


## Extra code reps (architecture mental models)

Python (4 blocks):
```python
# 1) Simple tokenizer sketch
import re

def tokens(src: str):
    for match in re.finditer(r"[a-zA-Z_]+|[0-9]+|\S", src):
        yield match.group(0)

print(list(tokens("a = 1 + 2")))
```
```python
# 2) AST node dataclasses
from dataclasses import dataclass

@dataclass
class Assign:
    name: str
    value: int

print(Assign("x", 1))
```
```python
# 3) visitor walk
class Visitor:
    def visit_assign(self, node: Assign):
        print(f"assign {node.name} -> {node.value}")

Visitor().visit_assign(Assign("x", 1))
```
```python
# 4) tiny eval
env = {}
code = [Assign("x", 3)]
for stmt in code:
    env[stmt.name] = stmt.value
print(env)
```

Rust (4 blocks):
```rust
// 1) Tokenizer-like split
fn tokens(src: &str) -> Vec<&str> {
    src.split_whitespace().collect()
}
```
```rust
// 2) AST struct
#[derive(Debug)]
struct Assign<'a> { name: &'a str, value: i32 }
```
```rust
// 3) Visitor trait
trait Visitor {
    fn visit_assign(&mut self, node: &Assign);
}
```
```rust
// 4) Eval loop
fn eval(code: &[Assign]) -> std::collections::HashMap<&str, i32> {
    let mut env = std::collections::HashMap::new();
    for stmt in code {
        env.insert(stmt.name, stmt.value);
    }
    env
}
```

KCL (4 blocks):
```kcl
# 1) Token split
src = "a = 1 + 2"
tokens = src.split()
print(tokens)
```
```kcl
# 2) Schema as AST node
schema Assign:
    name: str
    value: int

print(Assign{name="x", value=1})
```
```kcl
# 3) Visitor-like function
visit_assign = lambda node: print("assign", node.name, node.value)
visit_assign(Assign{name="x", value=1})
```
```kcl
# 4) Eval loop
env = {}
code = [Assign{name="x", value=3}]
for stmt in code:
    env[stmt.name] = stmt.value
print(env)
```
