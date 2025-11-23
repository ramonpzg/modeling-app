# Part 5: The KCL Standard Library: Your First Disappointment (Expanded)

You've learned the fundamentals. You know how to fail gracefully. Now it's time to get your hands dirty in the KCL codebase itself. In this part, we're not going to *pretend* to add a function to the standard library; we're going to walk through the *actual* process. It's less disappointing when you know what you're doing.

## A Tour of the Standard Library

The KCL standard library is where the fundamental building blocks of geometry are defined. You can find it in `rust/kcl-lib/src/std`. Each file corresponds to a different category of operations: `shapes.rs` for creating 2D shapes, `extrude.rs` for, well, extruding them, `math.rs` for mathematical functions, and so on.

The functions here are written in Rust, but they are exposed to the KCL environment. This is where the strongly-typed, compiled world of Rust meets the dynamically-typed, interpreted world of KCL.

## Our Goal: Adding a `regularPolygon` Function

Let's add a genuinely useful function: `regularPolygon`. It will create a regular polygon with a given number of sides, inscribed within a circle of a given radius.

This is a real function that could plausibly be in the standard library. The existing `polygon` function is more general, taking a list of points, so this is a good candidate for a new, convenient helper function.

### Step 1: Find the Right Home

Looking at the file list in `rust/kcl-lib/src/std`, the obvious place for a new 2D shape function is `shapes.rs`. We'll add our function there.

### Step 2: Write the Rust Function

Open `rust/kcl-lib/src/std/shapes.rs`. We'll add a new `async` function. The function signature will look something like this:

```rust
// In rust/kcl-lib/src/std/shapes.rs

/// Create a regular polygon with a specified number of sides.
pub async fn regular_polygon(exec_state: &mut ExecState, args: Args) -> Result<KclValue, KclError> {
    // ... implementation ...
}
```

Every standard library function takes an `&mut ExecState`, which holds the state of the KCL execution environment, and an `Args` struct, which contains the arguments passed from the KCL code. It returns a `Result<KclValue, KclError>`, where `KclValue` is the value that will be returned to the KCL environment.

The implementation will involve:
1.  Parsing the arguments from the `Args` struct (center point, radius, number of sides).
2.  Calculating the vertices of the polygon using some trigonometry.
3.  Sending modeling commands to the geometry engine to create the polygon.
4.  Returning the created shape as a `KclValue`.

Here is a plausible, albeit simplified, implementation:

```rust
// In rust/kcl-lib/src/std/shapes.rs, assuming other necessary `use` statements are present.

/// Create a regular polygon with a specified number of sides.
pub async fn regular_polygon(exec_state: &mut ExecState, args: Args) -> Result<KclValue, KclError> {
    let sketch_or_surface =
        args.get_unlabeled_kw_arg("sketchOrSurface", &RuntimeType::sketch_or_surface(), exec_state)?;
    let center = args.get_kw_arg("center", &RuntimeType::point2d(), exec_state)?;
    let radius: TyF64 = args.get_kw_arg("radius", &RuntimeType::length(), exec_state)?;
    let num_sides: TyF64 = args.get_kw_arg("numSides", &RuntimeType::count(), exec_state)?;

    if num_sides.n < 3.0 {
        return Err(KclError::new_type(KclErrorDetails::new(
            "A polygon must have at least 3 sides.".to_string(),
            vec![args.source_range],
        )));
    }

    let num_sides = num_sides.n as u64;
    let angle_step = 2.0 * std::f64::consts::PI / num_sides as f64;

    let vertices: Vec<[TyF64; 2]> = (0..num_sides)
        .map(|i| {
            let angle = angle_step * i as f64;
            [
                TyF64::new(center[0].n + radius.n * angle.cos(), center[0].ty),
                TyF64::new(center[1].n + radius.n * angle.sin(), center[1].ty),
            ]
        })
        .collect();

    // This is a simplification. A real implementation would send modeling commands
    // to draw the lines between vertices, similar to the `polygon` function.
    // For this tutorial, we'll just log the vertices.
    println!("Calculated vertices: {:?}", vertices);


    // For a real implementation, you would then use `inner_start_profile` and `ExtendPath`
    // commands to build the sketch, as seen in the `polygon` function in shapes.rs.
    // Then you would return Ok(KclValue::Sketch { ... }).
    // For now, we return an empty sketch to satisfy the type checker.

    let sketch = crate::std::sketch::inner_start_profile(sketch_or_surface, center, None, exec_state, args).await?;
    Ok(KclValue::Sketch{ value: Box::new(sketch) })
}
```

### Step 3: Expose it to KCL

Now, we need to tell the KCL compiler about our new function. We do this in `rust/kcl-lib/src/std/mod.rs`. This file is the module's root and contains a macro that builds the standard library.

Find the `stdlib!` macro call and add our new function to the list.

```rust
// In rust/kcl-lib/src/std/mod.rs

stdlib! {
    // ... other functions ...
    pub fn regularPolygon = shapes::regular_polygon;
    // ... other functions ...
}
```

The format is `pub fn <kclName> = <rust_module>::<rust_function_name>;`. By convention, KCL functions are camelCase, while Rust functions are snake_case.

### Step 4: Write a Test

How do we know this works? We write a *simulation test*. These tests live in `rust/kcl-lib/e2e/`. They execute a KCL file and compare the output to a snapshot.

The process is managed by a tool called `just`. First, you'd create a new test:

```bash
just new-sim-test my_regular_polygon_test
```

This creates a new directory `rust/kcl-lib/e2e/my_regular_polygon_test` with an `input.kcl` file. You would edit that file to call your new function:

```kcl
// In rust/kcl-lib/e2e/my_regular_polygon_test/input.kcl

const sketch = startSketchOn("XY")
const poly = regularPolygon(sketch, { center: [0, 0], radius: 10, numSides: 6 })
```

Then, you run the test and tell it to overwrite the snapshots.

```bash
just overwrite-sim-test my_regular_polygon_test
```

This will run your KCL code through the real geometry engine and generate a bunch of output files (`.snap` files) in the test directory. You should inspect these files to make sure the output is what you expected. If it is, you commit them along with your code.

This process gives you a high degree of confidence that your function not only compiles but also correctly interacts with the geometry engine.

This has been a whirlwind tour, but it's the real, unfiltered process of contributing to KCL. In the next part, we'll peel back another layer and look at the KCL parser, the part of the compiler that first reads your code.
