# Part 09 — Notebook kernel blueprint with runnable parallels

We translate kernel messaging ideas into small Python prototypes, Rust sketches, and KCL configs that describe the system.

## Message model
Python ZMQ sketch:
```python
import zmq, json
ctx = zmq.Context()
sock = ctx.socket(zmq.REP)
sock.bind("tcp://*:5555")
msg = sock.recv_json()
sock.send_json({"status":"ok","echo":msg})
```
Rust zmq (conceptual):
```rust
use zmq::Context;
fn main() {
    let ctx = Context::new();
    let sock = ctx.socket(zmq::REP).unwrap();
    sock.bind("tcp://*:5555").unwrap();
    let msg = sock.recv_msg(0).unwrap();
    sock.send(msg, 0).unwrap();
}
```
KCL describing endpoints:
```kcl
schema Endpoints:
    shell: str = "tcp://*:5555"
    iopub: str = "tcp://*:5556"

print(Endpoints{}.shell)
```
Python message struct:
```python
msg = {
    "header": {"msg_id": "1", "msg_type": "execute_request"},
    "content": {"code": "1 + 1"},
}
print(json.dumps(msg))
```
Rust struct representation:
```rust
use serde::{Serialize, Deserialize};
#[derive(Serialize, Deserialize, Debug)]
struct ExecuteRequest { code: String }
fn main() {
    let req = ExecuteRequest { code: "1 + 1".into() };
    println!("{}", serde_json::to_string(&req).unwrap());
}
```
KCL config describing a message:
```kcl
schema ExecuteRequest:
    code: str

req: ExecuteRequest = {code:"1 + 1"}

__main__:
    print(req.code)
```

## Execution flow
Python synchronous handler:
```python
def handle_execute(code: str) -> dict:
    try:
        result = eval(code)
        return {"status":"ok","result":result}
    except Exception as e:
        return {"status":"error","ename":type(e).__name__}

print(handle_execute("1+1"))
```
Rust handler sketch:
```rust
fn handle_execute(code: &str) -> serde_json::Value {
    match meval::eval_str(code) {
        Ok(v) => serde_json::json!({"status":"ok","result":v}),
        Err(e) => serde_json::json!({"status":"error","ename":e.to_string()}),
    }
}
```
KCL representation of execution outcome:
```kcl
schema ExecuteReply:
    status: str
    result?: any
    ename?: str

ok: ExecuteReply = {status:"ok", result:2}
err: ExecuteReply = {status:"error", ename:"ZeroDivisionError"}
```
Python streaming stdout:
```python
import sys
sys.stdout.write("hello")
```
Rust capturing output:
```rust
use std::io::Write;
fn main() {
    let mut buf = Vec::new();
    write!(&mut buf, "hello").unwrap();
    println!("{:?}", String::from_utf8(buf).unwrap());
}
```
KCL stdout is conceptual only:
```kcl
output: str = "hello"
```

## Kernel state
Python in-memory notebook session:
```python
class Session:
    def __init__(self):
        self.scope = {}
    def exec(self, code):
        exec(code, self.scope)
        return self.scope

s = Session()
print(s.exec("x=2"))
```
Rust state struct:
```rust
use std::collections::HashMap;
struct Session { scope: HashMap<String, String> }
impl Session {
    fn exec(&mut self, code: &str) { self.scope.insert("last".into(), code.into()); }
}
fn main() {
    let mut s = Session { scope: HashMap::new() };
    s.exec("x=2");
    println!("{:?}", s.scope);
}
```
KCL session as config snapshot:
```kcl
schema Session:
    scope: {str:str}

s: Session = {scope:{"x":"2"}}

__main__:
    print(s.scope)
```
Python history tracking:
```python
class History:
    def __init__(self):
        self.lines = []
    def add(self, code):
        self.lines.append(code)

h = History(); h.add("1+1"); print(h.lines)
```
Rust Vec-based history:
```rust
struct History { lines: Vec<String> }
impl History { fn add(&mut self, code: &str) { self.lines.push(code.into()); } }
fn main() {
    let mut h = History { lines: vec![] };
    h.add("1+1");
    println!("{:?}", h.lines);
}
```
KCL history config:
```kcl
history: [str] = ["1+1"]

__main__:
    print(history)
```

## Display data
Python mime bundle:
```python
bundle = {"text/plain": "hi", "text/html": "<b>hi</b>"}
print(bundle)
```
Rust mime bundle struct:
```rust
use std::collections::HashMap;
fn main() {
    let mut bundle = HashMap::new();
    bundle.insert("text/plain", "hi");
    bundle.insert("text/html", "<b>hi</b>");
    println!("{:?}", bundle);
}
```
KCL bundle config:
```kcl
bundle: {str:str} = {"text/plain":"hi", "text/html":"<b>hi</b>"}
```
Python plotting stub:
```python
import matplotlib.pyplot as plt
plt.plot([0,1],[0,1])
plt.savefig("plot.png")
```
Rust plot via plotters crate (conceptual):
```rust
fn main() {
    // build a chart and save to file using plotters
}
```
KCL referencing generated artifact:
```kcl
plot_path: str = "plot.png"
```

## Integration checklist
Python end-to-end kernel loop sketch:
```python
while True:
    req = sock.recv_json()
    reply = handle_execute(req["content"]["code"])
    sock.send_json(reply)
```
Rust outline:
```rust
fn main() {
    // recv zmq, parse to ExecuteRequest, call eval, send reply
}
```
KCL configuration describing ports and timeouts:
```kcl
schema KernelCfg:
    shell: str
    iopub: str
    hb: str
    timeout_ms: int

cfg: KernelCfg = {shell:"tcp://*:5555", iopub:"tcp://*:5556", hb:"tcp://*:5557", timeout_ms:1000}
```

## Exercises
1. Extend the Python handler to stream stdout and stderr separately and mirror the shape in Rust structs and KCL schemas.
2. Add execution_count tracking to all three representations.
3. Model comm_open/comm_msg in KCL configs, then prototype a Python handler stub.
4. Identify which parts of `kcl-lib` you would call from the kernel (parser + evaluator) and list the FFI boundary surface.


## Extra code reps (kernel messaging intuition)

Python (4 blocks):
```python
# 1) JSON message build
import json
msg = {"header": {"msg_type": "execute_request"}, "content": {"code": "print(1)"}}
print(json.dumps(msg))
```
```python
# 2) ZeroMQ send/recv skeleton
import zmq
ctx = zmq.Context()
sock = ctx.socket(zmq.PAIR)
# sock.bind(...) / sock.connect(...)
```
```python
# 3) asyncio stream echo
import asyncio

async def echo(reader, writer):
    data = await reader.readline()
    writer.write(data)
    await writer.drain()

# asyncio.start_server(echo, "127.0.0.1", 9999)
```
```python
# 4) ipykernel-like reply function
def reply(status: str, execution_count: int):
    return {"status": status, "execution_count": execution_count}

print(reply("ok", 1))
```

Rust (4 blocks):
```rust
// 1) serde_json message
use serde_json::json;
fn message() -> serde_json::Value {
    json!({"header": {"msg_type": "execute_request"}, "content": {"code": "print(1)"}})
}
```
```rust
// 2) zmq skeleton
// let ctx = zmq::Context::new();
// let socket = ctx.socket(zmq::PAIR).unwrap();
```
```rust
// 3) tokio TCP echo
// async fn echo(stream: TcpStream) { ... }
```
```rust
// 4) reply builder
#[derive(serde::Serialize)]
struct Reply<'a> { status: &'a str, execution_count: u32 }
```

KCL (4 blocks):
```kcl
# 1) JSON-like map
msg = {"header": {"msg_type": "execute_request"}, "content": {"code": "print(1)"}}
print(msg)
```
```kcl
# 2) Socket config placeholder
socket_cfg = {"transport": "tcp", "endpoint": "127.0.0.1:9999"}
print(socket_cfg)
```
```kcl
# 3) Simulated echo
input_line = "print(1)"
print(input_line)
```
```kcl
# 4) Reply map builder
reply = lambda status, count: {"status": status, "execution_count": count}
print(reply("ok", 1))
```
