# Part 1: The Obligatory "Hello, World!" (Now with more substance)

So, you want to learn Rust and KCL. You've come to the right place. I've been told I'm an excellent teacher, mostly by myself. We're going to get you from zero to a KCL contributor. It will be a painful, yet rewarding, journey. Mostly painful.

Let's start with the basics. Rust is a systems programming language that boasts about its speed, safety, and concurrency. It's the programming language equivalent of a CrossFit enthusiast: it's very effective, but it will never shut up about it. Its most famous feature is the "borrow checker," which is a compiler-enforced set of rules that prevents you from shooting yourself in the foot with memory management. It's like having a very pedantic, very shouty guardian angel.

## Your First Rust Program: Cargo Culting

In the grand tradition of programming, we begin with "Hello, World!". Open your terminal. We're not just writing a file; we're creating a *project*.

```bash
cargo new hello_rust
cd hello_rust
```

`cargo` is Rust's command-line tool that handles project creation, dependency management, building, testing, and more. It's the Swiss Army knife you'll actually use. This command creates a new directory called `hello_rust` containing a simple Rust project.

Let's look at what it made:

```
.
├── Cargo.toml
└── src
    └── main.rs
```

-   `src/main.rs`: This is where your code lives.
-   `Cargo.toml`: This is the project's configuration file, or "manifest."

### The Manifest: `Cargo.toml`

If you're from Python, `Cargo.toml` is like `pyproject.toml` or `requirements.txt`. It's where you list your project's metadata and dependencies. Open it up.

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
```

It's pretty self-explanatory. The `[dependencies]` section is where you'd add other Rust libraries (called "crates") that you want to use. We'll get to that later. Don't worry, dependency hell is a universal constant.

### The Code: `src/main.rs`

Now, let's look at the code in `src/main.rs`.

```rust
fn main() {
    println!("Hello, world!");
}
```

Let's dissect this:

-   `fn main()`: This declares a function named `main`. It's the entry point for all executable Rust programs. No `if __name__ == "__main__":` boilerplate here. Rust likes to get straight to the point.
-   `println!("Hello, world!");`: This prints text to the console. The `!` signifies that `println` is a *macro*, not a regular function. What's the difference? Macros write code for you at compile time. It's a form of metaprogramming. For now, just think of them as super-powered functions and be appropriately wary.

## Building and Running: Debug vs. Release

Now, let's run this masterpiece.

```bash
cargo run
```

You'll see some output about compiling, and then `Hello, world!`. `cargo run` compiles and runs your code. This creates a "debug" build, which is full of extra information that helps with debugging. It's slower, but it holds your hand.

When you're ready to ship your code to the masses (or just want it to run fast), you'll want a "release" build.

```bash
cargo run --release
```

This will take longer to compile, but the resulting executable will be much faster. It's like the difference between your code wearing pajamas and a full suit of armor.

After you build, you'll also notice a new file: `Cargo.lock`. This file contains the exact versions of all your dependencies. It's like Python's `poetry.lock` or `Pipfile.lock`, ensuring your builds are reproducible. Don't touch it. Cargo manages it for you.

## Your First KCL Program: Actually Making Something

KCL (KittyCAD Language) is a declarative language for defining geometry. You don't write steps to *do* something; you describe the *thing* you want.

Our first KCL program won't be a useless comment. We'll make a point.

```kcl
// This program defines a single point in 2D space.
const myPoint = {
  x: 10,
  y: 20
}
```

Let's break it down:
- `//`: This is a comment, just like in Rust or JavaScript.
- `const myPoint = ...`: This declares a constant named `myPoint`. A constant is a value that cannot be changed.
- `{ x: 10, y: 20 }`: This is an object literal that defines a point with an x-coordinate of 10 and a y-coordinate of 20. KCL infers that this is a 2D point.

This is a complete, valid KCL program. It doesn't *do* anything with the point yet, but it defines a piece of geometry. We could later use this point to, say, start a line or be the center of a circle.

### The Python Equivalent

In a Python-based CAD library like CadQuery, you might do something like this:

```python
import cadquery as cq

# A point is often just a tuple or a vector object
my_point = (10, 20)

# In CadQuery, you'd typically start a chain of operations
# result = cq.Workplane("XY").center(10, 20)
```

The KCL version is more about describing the data, while the Python version is more about a sequence of commands.

Now you've got your feet wet. In the next part, we'll dive into Rust's type system and control flow, and I'll properly introduce you to the borrow checker. Prepare yourself. It's not gentle.
