# Part 2: Types, Control Flow, and Why Rust Thinks You're Irresponsible (Expanded)

Welcome back. If Part 1 was dipping your toes in, this is the part where I push you into the deep end. We're going to cover Rust's fundamental building blocks: its type system and control flow. More importantly, we're going to properly confront the concept of "ownership," which is Rust's way of preventing you from making a mess of memory.

## Rust's Type System: A Rigid Framework

Coming from Python's dynamic typing is like leaving a comfortable, messy apartment for a sterile, minimalist one. Rust is statically typed, meaning the type of every variable must be known at compile time. This feels restrictive, but it eliminates a whole class of runtime errors.

### Scalar Types: The Atoms of Rust

Scalar types represent a single value. Rust has four primary scalar types:

*   **Integers**: Numbers without a fractional component. They come in signed (`i`) and unsigned (`u`) variants, and in sizes of 8, 16, 32, 64, and 128 bits. For example, `u32` is a 32-bit unsigned integer, and `i64` is a 64-bit signed integer. The default is `i32`.
*   **Floating-Point Numbers**: Numbers with a fractional component. They come in `f32` and `f64` sizes. The default is `f64`.
*   **Booleans**: The `bool` type, with values `true` or `false`.
*   **Characters**: The `char` type, representing a single Unicode Scalar Value.

```rust
let an_integer: u32 = 42;
let a_float = 3.14; // Defaults to f64
let is_rust_fun = true;
let a_char = '😻';
```

### Compound Types: Grouping Values

Rust also has ways to group multiple values into one type.

*   **Tuples**: A fixed-size collection of values of different types. Think of them as immutable, fixed-length Python tuples.
    ```rust
    let my_tuple: (i32, f64, char) = (500, 6.4, 'a');
    let five_hundred = my_tuple.0; // Access by index
    ```
*   **Arrays**: A fixed-size collection of values of the *same* type.
    ```rust
    let my_array: [i32; 5] = [1, 2, 3, 4, 5];
    let first_element = my_array[0];
    ```
    Arrays are less flexible than Python lists or Rust's `Vec<T>` (which we'll cover later), because their size cannot change.

## Ownership: The Stack, the Heap, and the Borrow Checker

This is the hard part. To understand ownership, you need to understand where your data lives in memory.

*   **The Stack**: Fast, ordered, and small. It works like a stack of plates. When you call a function, its local variables are pushed onto the stack. When the function returns, they're popped off. All data on the stack must have a known, fixed size.
*   **The Heap**: Slower, less organized, and large. When you need to store data of an unknown or dynamic size (like a string that can grow), you allocate memory on the heap. This is more expensive because the program has to find a free spot and then return a *pointer* to that spot.

In Python, everything is a pointer to a heap-allocated object. In Rust, you have a choice. Scalar types, tuples, and arrays of fixed-size types live on the stack. They are cheap to copy.

```rust
let x = 5; // An i32, lives on the stack
let y = x; // The value 5 is copied. No problem.
```

But what about types like `String`, which can grow? They are stored on the heap. The `String` variable on the stack stores a pointer to the heap data, along with its length and capacity.

This is where ownership comes in. Rust says: **every value on the heap can only have one owner.**

```rust
let s1 = String::from("hello"); // s1 owns a string on the heap
let s2 = s1;                   // Ownership of the string is MOVED to s2
// s1 is no longer valid. The compiler will stop you here.
// println!("{}", s1); // COMPILE ERROR!
```

This prevents a "double free" error, where two variables might try to free the same memory when they go out of scope. Instead of copying the potentially large data on the heap (which is slow), Rust just moves the "title of ownership."

### Borrowing: The Art of Sharing

What if you just want a function to *read* a value without taking ownership? You can "borrow" it by creating a *reference*.

```rust
fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
} // s goes out of scope, but since it doesn't own the string, nothing is freed.

let s1 = String::from("hello");
let len = calculate_length(&s1); // We pass a reference to s1
println!("The length of '{}' is {}.", s1, len); // s1 is still valid here!
```

You can also have mutable references, but with a big rule: **you can have either one mutable reference OR any number of immutable references, but not both at the same time.** This is how Rust prevents data races at compile time.

## Control Flow: More Than Just `if`

Rust's control flow is powerful because constructs like `if` are *expressions*, meaning they return a value.

```rust
let number = 6;
let result = if number % 4 == 0 {
    "divisible by 4"
} else if number % 3 == 0 {
    "divisible by 3"
} else {
    "not divisible by 4 or 3"
}; // Note the semicolon here, for the `let` statement.
```

This is much cleaner than the equivalent Python code.

The `for` loop in Rust is used to iterate over any `Iterator`.

```rust
let a = [10, 20, 30, 40, 50];

for element in a {
    println!("the value is: {}", element);
}

// For a Python-like range:
for number in (1..4).rev() { // 1 to 4 (exclusive), reversed
    println!("{}!", number);
}
```

## KCL: Control Flow in Declarative Land

KCL is mostly declarative, but it does have some control flow constructs. For instance, you can use loops to generate multiple objects.

```kcl
const points = []
for i from 0 to 10 {
  points.push({x: i, y: i * i})
}

const lines = []
for i from 0 to 9 {
  lines.push(line(points[i], points[i+1]))
}
```
This example generates a series of points for a parabola and then connects them with lines. It's a blend of declarative geometry with imperative logic.

We've covered a lot of ground here. Take a moment to let the horror of the ownership rules sink in. In the next part, we'll look at how to structure your data with `structs` and `enums`.
