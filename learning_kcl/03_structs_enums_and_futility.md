# Part 3: Structs, Enums, and the Futility of Existence (Expanded)

You're still here? Excellent. Now we move from wrestling with the borrow checker to the more refined art of defining our own data structures. With `struct`s and `enum`s, you can create your own types. It's like playing God, but the universe is your compiler, and it's constantly telling you you're wrong.

## Structs: Building Your Own Blueprints

A `struct` is a collection of fields. Think of it as a custom type that bundles together related data. It's the Rust equivalent of a Python class, but without the baggage of inheritance.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}
```

This defines a `Rectangle` with two fields. But data is useless without behavior. To add methods, we use an `impl` (implementation) block.

```rust
impl Rectangle {
    // This is a method. It takes an immutable reference to self.
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // We can also have methods that take a mutable reference.
    fn scale(&mut self, factor: f64) {
        self.width = (self.width as f64 * factor) as u32;
        self.height = (self.height as f64 * factor) as u32;
    }

    // This is an "associated function," not a method, because it doesn't take `self`.
    // It's often used for constructors.
    fn new(width: u32, height: u32) -> Rectangle {
        Rectangle { width, height }
    }
}

let mut rect = Rectangle::new(30, 50);
println!("The area of the rectangle is {} square pixels.", rect.area());
rect.scale(1.5);
```

### The Python Equivalent

In Python, this would be a class.

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def scale(self, factor):
        self.width = int(self.width * factor)
        self.height = int(self.height * factor)

rect = Rectangle(30, 50)
print(f"The area is {rect.area()}")
```

The concepts are similar, but Rust's `impl` blocks are a more explicit way of separating data definition from behavior.

## Enums: One Type, Many Possibilities

Enums (enumerations) are a way of saying a value can be one of a number of possible variants. But in Rust, enums are supercharged. You can attach data directly to each variant.

Let's model a geometric primitive. It could be a point, a line, or a circle.

```rust
struct Point { x: f64, y: f64 }

enum GeometricPrimitive {
    Point(Point),
    Line(Point, Point),
    Circle { center: Point, radius: f64 },
}
```

Here, `GeometricPrimitive` can be one of three things, and each variant carries different data. This is far more powerful than enums in languages like C or the anemic `Enum` in Python.

## `match`: Handling All the Cases

The true power of enums is unlocked with `match`. `match` is an expression that lets you compare a value against a series of patterns and execute code based on which pattern it matches. The compiler guarantees that your `match` is *exhaustive*—you must handle every possible variant.

```rust
fn print_primitive(primitive: &GeometricPrimitive) {
    match primitive {
        GeometricPrimitive::Point(p) => {
            println!("A point at ({}, {})", p.x, p.y);
        }
        GeometricPrimitive::Line(start, end) => {
            println!("A line from ({}, {}) to ({}, {})", start.x, start.y, end.x, end.y);
        }
        GeometricPrimitive::Circle { center, radius } => {
            println!("A circle with radius {} centered at ({}, {})", radius, center.x, center.y);
        }
    }
}
```

The `match` statement not only identifies the variant but also lets you *destructure* the data inside it. This is a huge win for writing safe, readable code.

## KCL: Composing Geometry

In KCL, you compose geometry by defining objects and then using them to build more complex objects. KCL's strength is in how it lets you describe these relationships declaratively.

Let's build a simple wheel.

```kcl
// Define some parameters
const wheelRadius = 10
const boltCircleRadius = 7
const numBolts = 5
const boltRadius = 1

// The main body of the wheel
const wheelBody = circle(0, 0, wheelRadius)

// Create the bolt holes
const boltHoles = []
for i from 0 to numBolts {
  const angle = 360 / numBolts * i
  const x = boltCircleRadius * cos(angle)
  const y = boltCircleRadius * sin(angle)
  boltHoles.push(circle(x, y, boltRadius))
}

// Subtract the bolt holes from the wheel body
const finalWheel = wheelBody - boltHoles
```

This KCL snippet defines a wheel, creates a pattern of bolt holes using a loop and some trigonometry, and then subtracts the holes from the main body. This composition of declarative definitions is the core of KCL's power.

Now that you can create your own data structures, you're starting to look like a real programmer. In the next section, we'll cover error handling, so you can learn how to fail gracefully.
