# Part 06 — Async foundations and FFI hooks

We need async to talk to kernels and FFI to bridge KCL with Python. Examples show event loops and minimal bindings.

## Async basics
Python asyncio:
```python
import asyncio

async def fetch(x):
    await asyncio.sleep(0.1)
    return x * 2

async def main():
    results = await asyncio.gather(fetch(1), fetch(2))
    print(results)

asyncio.run(main())
```
Rust async with tokio:
```rust
use tokio::time::{sleep, Duration};

async fn fetch(x: i32) -> i32 {
    sleep(Duration::from_millis(100)).await;
    x * 2
}

#[tokio::main]
async fn main() {
    let res = futures::future::join(fetch(1), fetch(2)).await;
    println!("{:?}", res);
}
```
KCL does not have async; treat it as configuration. Simulate dependency ordering:
```kcl
schema Job:
    input: int
    output: int = input * 2

jobs: [Job] = [Job{input:1}, Job{input:2}]
outputs = [j.output for j in jobs]

__main__:
    print(outputs)
```
Python asyncio cancellation:
```python
async def slow():
    await asyncio.sleep(1)
    return "done"

async def main():
    task = asyncio.create_task(slow())
    await asyncio.sleep(0.1)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("cancelled")

asyncio.run(main())
```
Rust cancellation via `select!`:
```rust
use tokio::select;
use tokio::time::{sleep, Duration};

async fn slow() -> &'static str {
    sleep(Duration::from_secs(1)).await;
    "done"
}

#[tokio::main]
async fn main() {
    let task = slow();
    let timeout = sleep(Duration::from_millis(100));
    select! {
        res = task => println!("{}", res),
        _ = timeout => println!("cancelled"),
    }
}
```
KCL alternative: choose between options:
```kcl
schema Choice:
    fast: bool
    result: str = "done" if fast else "cancelled"

print(Choice{fast:False}.result)
```

## Channels and messaging
Python asyncio queue:
```python
import asyncio

async def producer(q):
    for i in range(3):
        await q.put(i)
    await q.put(None)

async def consumer(q):
    while True:
        item = await q.get()
        if item is None:
            break
        print(item)

async def main():
    q = asyncio.Queue()
    await asyncio.gather(producer(q), consumer(q))

asyncio.run(main())
```
Rust tokio mpsc:
```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(10);
    let producer = tokio::spawn(async move {
        for i in 0..3 {
            tx.send(i).await.unwrap();
        }
    });
    while let Some(v) = rx.recv().await {
        println!("{}", v);
    }
    producer.await.unwrap();
}
```
KCL static message list:
```kcl
messages: [int] = [0,1,2]

__main__:
    for m in messages:
        print(m)
```
Python trio alternative:
```python
import trio

async def producer(send_channel):
    async with send_channel:
        for i in range(3):
            await send_channel.send(i)

async def consumer(receive_channel):
    async with receive_channel:
        async for value in receive_channel:
            print(value)

async def main():
    send, recv = trio.open_memory_channel(10)
    await trio.gather(producer(send), consumer(recv))

trio.run(main)
```
Rust async-std example:
```rust
use async_std::task;
use async_channel;

fn main() {
    task::block_on(async {
        let (s, r) = async_channel::bounded(10);
        let sender = task::spawn(async move {
            for i in 0..3 {
                s.send(i).await.unwrap();
            }
        });
        while let Ok(v) = r.recv().await {
            println!("{}", v);
            if v == 2 { break; }
        }
        sender.await;
    });
}
```
KCL mapping messages to actions:
```kcl
msgs = ["run", "stop"]
actions = [m.upper() for m in msgs]

__main__:
    print(actions)
```

## FFI with Python
Python calling Rust via pyo3 (concept sketch):
```python
# Cargo.toml will have pyo3 with features = ["extension-module"]
# In Rust you expose #[pyfunction]; here we call the built wheel
import rust_kcl
print(rust_kcl.eval_kcl("1 + 2"))
```
Rust pyo3 stub:
```rust
use pyo3::prelude::*;

#[pyfunction]
fn eval_kcl(expr: String) -> PyResult<String> {
    Ok(format!("evaluated {expr}"))
}

#[pymodule]
fn rust_kcl(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(eval_kcl, m)?)?;
    Ok(())
}
```
KCL being evaluated:
```kcl
schema Expr:
    val: int = 1 + 2

__main__:
    print(Expr{}.val)
```
Python using cffi to call Rust-generated C ABI:
```python
from cffi import FFI
ffi = FFI()
ffi.cdef("int add(int a, int b);")
lib = ffi.dlopen("./target/release/libadd.so")
print(lib.add(1,2))
```
Rust exposing C ABI:
```rust
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 { a + b }
```
KCL call result consumed as config:
```kcl
result: int = 1 + 2  # imagine fed from FFI

__main__:
    print(result)
```

## WASM and sandboxing
Python calling wasm via wasmtime:
```python
from wasmtime import Store, Module, Instance
store = Store()
module = Module.from_file(store.engine, "kcl.wasm")
instance = Instance(store, module, [])
run = instance.exports(store)["run"]
print(run(store))
```
Rust building wasm with `wasm-bindgen`:
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn run() -> String {
    "hello from wasm".into()
}
```
KCL compiled to WASM via existing package (conceptual):
```kcl
schema Program:
    message: str = "hello from wasm"

__main__:
    print(Program{}.message)
```
Python comparing sync vs async latency measurement:
```python
import time, asyncio
async def run():
    start = time.time()
    await asyncio.sleep(0.1)
    return time.time() - start
print(asyncio.run(run()))
```
Rust timing:
```rust
use std::time::Instant;
use tokio::time::sleep;
use std::time::Duration;

#[tokio::main]
async fn main() {
    let start = Instant::now();
    sleep(Duration::from_millis(100)).await;
    println!("{:?}", start.elapsed());
}
```
KCL static timing value:
```kcl
time_spent_ms: int = 100

__main__:
    print(time_spent_ms)
```

## Exercises
1. Build a tiny pyo3 Rust module that adds two numbers and import it from Python. Then imagine replacing it with a call into `kcl-lib` evaluator.
2. Sketch how the async channel example maps to Jupyter kernel messaging (REQ/REP for execute, PUB for IOPub).
3. Replace the tokio `select!` sample with `futures::select` and note the syntactic differences.
4. Create a KCL configuration describing async tasks and dependencies, even though KCL itself is sync, to model your kernel pipeline.


## Extra code reps (async and FFI flavored)

Python (4 blocks):
```python
# 1) asyncio gather
import asyncio

async def task(n):
    await asyncio.sleep(0.1)
    return n * 2

async def main():
    results = await asyncio.gather(*(task(i) for i in range(3)))
    print(results)

asyncio.run(main())
```
```python
# 2) async context manager
class Resource:
    async def __aenter__(self):
        print("enter")
        return self
    async def __aexit__(self, *exc):
        print("exit")

async def use():
    async with Resource():
        print("work")

asyncio.run(use())
```
```python
# 3) ctypes call to C sqrt
import ctypes, math
libm = ctypes.CDLL(None)
libm.sqrt.argtypes = [ctypes.c_double]
libm.sqrt.restype = ctypes.c_double
print(libm.sqrt(9.0))
```
```python
# 4) multiprocessing queue (message passing)
from multiprocessing import Process, Queue

def worker(q: Queue):
    q.put("hi")

q = Queue()
p = Process(target=worker, args=(q,))
p.start(); p.join()
print(q.get())
```

Rust (4 blocks):
```rust
// 1) async executor (tokio)
#[tokio::main]
async fn main() {
    let handles = (0..3).map(|n| async move { n * 2 });
    let results: Vec<i32> = futures::future::join_all(handles).await;
    println!("{:?}", results);
}
```
```rust
// 2) async drop with RAII
struct Resource;
impl Drop for Resource {
    fn drop(&mut self) { println!("exit"); }
}
```
```rust
// 3) FFI to C sqrt
extern "C" {
    fn sqrt(input: f64) -> f64;
}

fn call_sqrt(x: f64) -> f64 {
    unsafe { sqrt(x) }
}
```
```rust
// 4) Channel message passing
use std::sync::mpsc;
fn message() {
    let (tx, rx) = mpsc::channel();
    std::thread::spawn(move || tx.send("hi").unwrap());
    println!("{}", rx.recv().unwrap());
}
```

KCL (4 blocks):
```kcl
# 1) async-like via task list (conceptual)
jobs = [n * 2 for n in range(3)]
print(jobs)
```
```kcl
# 2) RAII-style cleanup using schema lifecycle
schema Resource:
    enter = lambda : print("enter")
    exit = lambda : print("exit")

r = Resource{}
r.enter()
r.exit()
```
```kcl
# 3) FFI via builtin modules (placeholder)
# import("ffi/sqrt.k") # if available in env
```
```kcl
# 4) Message passing via list accumulation
msgs = []
msgs = msgs + ["hi"]
print(msgs[0])
```
