# Part 8: Advanced KCL Features (Sketches, Pipes, and Parametric Design)

You've learned the language. You've learned the stdlib. Now you'll understand what makes KCL a CAD language. Sketches, constraints, pipes, and parametric relationships are where KCL shines.

CAD is about intent. You don't just say "draw a line here." You say "this line is parallel to that one" or "this circle's diameter equals that rectangle's width." KCL captures intent with code.

## The Pipe Operator: More Than Syntax Sugar

You've seen the pipe operator:

```kcl
sketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [10, 0])
  |> line(to = [10, 10])
  |> close()
```

This is equivalent to:

```kcl
sketch = close(
  line(
    line(
      startProfile(
        startSketchOn("XY"),
        at = [0, 0]
      ),
      to = [10, 0]
    ),
    to = [10, 10]
  )
)
```

The pipe makes it readable. But it's more than that. Each function in the chain transforms the value. The type changes as you go:

1. `startSketchOn("XY")` → `Plane`
2. `startProfile(...)` → `SketchSurface`
3. `line(...)` → `SketchSurface` (with one more segment)
4. `line(...)` → `SketchSurface` (with two segments)
5. `close()` → `Sketch` (closed profile)

The type system tracks this. You can't call `extrude()` on an open profile. The compiler (well, the runtime type checker) stops you.

In Python, you'd use method chaining:

```python
sketch = (
    start_sketch_on("XY")
    .start_profile(at=[0, 0])
    .line(to=[10, 0])
    .line(to=[10, 10])
    .close()
)
```

But Python's version is just method calls. KCL's pipe operator is first-class syntax. The parser understands it. The AST represents it explicitly.

## How Pipes Work Internally

The parser transforms `x |> f(y)` into `f(x, y)`. The AST node:

```rust
pub enum Expr {
    PipeExpression(Box<PipeExpression>),
    // ... other variants
}

pub struct PipeExpression {
    pub left: Node<Expr>,
    pub right: Node<Expr>,  // Must be a call expression
}
```

The executor evaluates the left side, then passes it as the first argument to the right side:

```rust
async fn execute_pipe(&mut self, pipe: &PipeExpression) -> Result<KclValue, KclError> {
    let left_value = self.execute_expr(&pipe.left).await?;

    // Right side must be a function call
    match &pipe.right.inner {
        Expr::CallExpressionKw(call) => {
            // Add left_value as first argument
            let mut args = call.args.clone();
            args.insert(0, left_value);

            self.execute_call(&call.function, args).await
        }
        _ => Err(KclError::type_error("Pipe right side must be a function call")),
    }
}
```

This is simple but powerful. It makes sequential operations readable without nesting.

## Sketch Blocks: Special Syntax for CAD

KCL has special syntax for sketches:

```kcl
sketch001 = startSketchOn("XY")
  |> startProfile(at = [0, 0], to = [10, 0])
  |> line(to = [10, 10])
  |> line(to = [0, 10])
  |> close()
```

The `startProfile` function begins a sketch path. Each `line` adds a segment. `close` finishes the profile.

But there's more. KCL tracks the current position as you build:

```kcl
sketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [10, 0])        // From [0,0] to [10,0]
  |> line(to = [10, 10])       // From [10,0] to [10,10]
  |> line(to = [0, 10])        // From [10,10] to [0,10]
  |> close()                    // From [0,10] back to [0,0]
```

The `to` parameter is absolute. But you can use relative:

```kcl
sketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line([10, 0])             // Relative: move +10 in X
  |> line([0, 10])             // Relative: move +10 in Y
  |> line([-10, 0])            // Relative: move -10 in X
  |> close()
```

The function infers from the argument type. `line([10, 0])` is relative. `line(to = [10, 0])` is absolute.

In Python, you'd need explicit methods:

```python
sketch = (
    start_sketch_on("XY")
    .start_profile(at=[0, 0])
    .line_to([10, 0])           # Absolute
    .line_relative([0, 10])     # Relative
    .line_to([0, 10])
    .close()
)
```

KCL's version is terser and the intent is clearer.

## Tags: Naming Geometry for Later Reference

KCL lets you tag geometry:

```kcl
sketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [10, 0], $tag = "bottom")
  |> line(to = [10, 10], $tag = "right")
  |> line(to = [0, 10])
  |> close()

// Later, reference the tagged edge
fillet(sketch, $tag = "bottom", radius = 2)
```

Tags are first-class values. The `$` prefix indicates a tag reference.

Under the hood:

```rust
pub enum Expr {
    TagDeclarator(Box<TagDeclarator>),
    // ... other variants
}

pub struct TagDeclarator {
    pub name: String,
    pub value: Node<Expr>,
}
```

The executor stores tagged values in a separate table:

```rust
pub struct ExecState {
    pub tags: HashMap<String, KclValue>,
    // ... other fields
}
```

When you reference `$tag`, it looks up the value.

In Python, you'd use a dict:

```python
tags = {}

sketch = start_sketch_on("XY").start_profile(at=[0, 0])
tags["bottom"] = sketch.line(to=[10, 0])
tags["right"] = sketch.line(to=[10, 10])

# Later
fillet(sketch, edge=tags["bottom"], radius=2)
```

KCL makes this syntax-level, integrated into the language.

## Parametric Design: Variables and Constraints

The power of CAD programming is parametric design. Change one variable, the whole model updates:

```kcl
// Parameters
width = 100
height = 50
thickness = 10
filletRadius = 5

// Derived values
area = width * height
volume = area * thickness

// Geometry
baseSketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [width, 0])
  |> line(to = [width, height])
  |> line(to = [0, height])
  |> close()

solid = extrude(baseSketch, length = thickness)
  |> fillet(edges = all, radius = filletRadius)
```

Change `width = 100` to `width = 200`, the whole model scales. This is parametric modeling.

KCL tracks dependencies implicitly. When `width` changes, `area` recalculates, then `volume`, then the geometry. The engine recomputes the model.

In Python (non-CAD), you'd recalculate manually:

```python
class ParametricModel:
    def __init__(self):
        self.width = 100
        self.height = 50
        self.thickness = 10

    @property
    def area(self):
        return self.width * self.height

    @property
    def volume(self):
        return self.area * self.thickness

    def build(self):
        # Build geometry using current parameters
        pass

model = ParametricModel()
model.width = 200  # Change parameter
model.build()      # Rebuild
```

KCL's version is declarative. You state relationships, the system maintains them.

## Constraints: Geometric Relationships

KCL supports geometric constraints (experimental):

```kcl
sketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [10, 0], $tag = "bottom")
  |> line(to = [10, 10], $tag = "right")
  |> line(to = [0, 10], $tag = "top")
  |> close($tag = "left")
  |> constrain("bottom", parallel_to = "top")
  |> constrain("left", perpendicular_to = "bottom")
  |> constrain("right", length = 10)
```

Constraints are declarative. "This edge is parallel to that edge." The solver adjusts geometry to satisfy constraints.

Under the hood, constraints are sent to the engine as separate commands:

```rust
pub enum ModelingCmd {
    // ... geometry commands
    Constrain {
        entity1: EntityId,
        entity2: Option<EntityId>,
        constraint_type: ConstraintType,
    },
}

pub enum ConstraintType {
    Parallel,
    Perpendicular,
    Equal,
    FixedLength(f64),
    FixedAngle(f64),
    // ... more constraints
}
```

The engine's constraint solver adjusts the geometry to satisfy all constraints simultaneously.

In traditional CAD software, you apply constraints with UI clicks. In KCL, you write them as code:

```python
# Hypothetical Python CAD library
sketch = Sketch("XY")
bottom = sketch.line([0, 0], [10, 0])
right = sketch.line([10, 0], [10, 10])
top = sketch.line([10, 10], [0, 10])
left = sketch.line([0, 10], [0, 0])

sketch.add_constraint(Parallel(bottom, top))
sketch.add_constraint(Perpendicular(left, bottom))
sketch.add_constraint(FixedLength(right, 10))

sketch.solve()
```

KCL integrates constraints into the language.

## Extrude, Revolve, Sweep: 3D Operations

Once you have a 2D sketch, you create 3D solids:

**Extrude**: Pull a profile along a straight line:

```kcl
solid = extrude(sketch, length = 50)
```

**Revolve**: Rotate a profile around an axis:

```kcl
solid = revolve(sketch, axis = "Y", angle = 360degrees)
```

**Sweep**: Pull a profile along a path:

```kcl
path = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [10, 0])
  |> arc(to = [20, 10], radius = 10)

profile = startSketchOn("XZ")
  |> circle(center = [0, 0], radius = 2)

solid = sweep(profile, path)
```

Each operation sends commands to the engine:

```rust
pub async fn extrude(args: Args, exec_state: &mut ExecState) -> Result<KclValue, KclError> {
    let sketch = args.get_sketch("profile")?;
    let length = args.get_number("length")?;

    let response = exec_state
        .engine_connection
        .send_modeling_cmd(ModelingCmd::Extrude {
            profile_id: sketch.id,
            distance: length,
            cap: true,
        })
        .await?;

    Ok(KclValue::Solid {
        id: response.solid_id,
        sketch_id: sketch.id,
        // ... more fields
    })
}
```

The engine computes the 3D geometry and returns IDs. KCL wraps them in values for further operations.

## Boolean Operations: Combine Solids

Once you have multiple solids, you combine them:

```kcl
box1 = extrude(rect1, length = 10)
box2 = extrude(rect2, length = 10)

// Union (combine)
combined = union(box1, box2)

// Subtract (cut)
result = subtract(box1, box2)

// Intersect
overlap = intersect(box1, box2)
```

These are standard CSG (Constructive Solid Geometry) operations.

In Python (using a hypothetical CAD library):

```python
box1 = extrude(rect1, length=10)
box2 = extrude(rect2, length=10)

combined = box1.union(box2)
result = box1.subtract(box2)
overlap = box1.intersect(box2)
```

Same operations, but KCL's version is functional (returns new values) rather than mutating objects.

## Fillet and Chamfer: Edge Modifications

Fillets round edges. Chamfers bevel them.

```kcl
solid = extrude(sketch, length = 10)
  |> fillet(edges = all, radius = 2)

solid2 = extrude(sketch2, length = 10)
  |> chamfer(edges = [edge1, edge2], distance = 1)
```

These modify the solid by rounding or beveling specified edges.

```rust
pub async fn fillet(args: Args, exec_state: &mut ExecState) -> Result<KclValue, KclError> {
    let solid = args.get_solid("solid")?;
    let edges = args.get_edges("edges")?;
    let radius = args.get_number("radius")?;

    let response = exec_state
        .engine_connection
        .send_modeling_cmd(ModelingCmd::Fillet {
            solid_id: solid.id,
            edge_ids: edges.iter().map(|e| e.id).collect(),
            radius,
        })
        .await?;

    Ok(KclValue::Solid {
        id: response.solid_id,
        // ... more fields
    })
}
```

The engine modifies the geometry and returns the updated solid.

## Patterns: Replicate Geometry

You can pattern (copy) features:

```kcl
// Linear pattern
solid = extrude(sketch, length = 10)
  |> linearPattern(direction = [1, 0, 0], count = 5, spacing = 20)

// Circular pattern
solid2 = extrude(sketch2, length = 10)
  |> circularPattern(axis = "Z", count = 8, angle = 360degrees)
```

These create multiple copies of the feature.

In Python:

```python
solid = extrude(sketch, length=10)
copies = []
for i in range(5):
    copy = solid.translate([i * 20, 0, 0])
    copies.append(copy)

result = union(*copies)
```

KCL's version is declarative. The engine handles the copying and combining.

## Transforms: Move, Rotate, Scale

You can transform geometry:

```kcl
solid = extrude(sketch, length = 10)
  |> translate([10, 20, 30])
  |> rotate(axis = "Z", angle = 45degrees)
  |> scale(factor = 1.5)
```

Each transform returns a new solid (functional, not mutating).

```rust
pub async fn translate(args: Args, exec_state: &mut ExecState) -> Result<KclValue, KclError> {
    let solid = args.get_solid("solid")?;
    let offset = args.get_number_array("offset")?;

    let response = exec_state
        .engine_connection
        .send_modeling_cmd(ModelingCmd::Translate {
            solid_id: solid.id,
            offset: [offset[0], offset[1], offset[2]],
        })
        .await?;

    Ok(KclValue::Solid {
        id: response.solid_id,
        // ... more fields
    })
}
```

## Putting It All Together: A Complete Parametric Model

```kcl
// Parameters
baseWidth = 100
baseHeight = 50
baseThickness = 10
holeRadius = 5
holeOffset = 10
filletRadius = 3

// Base sketch
baseSketch = startSketchOn("XY")
  |> startProfile(at = [0, 0])
  |> line(to = [baseWidth, 0])
  |> line(to = [baseWidth, baseHeight])
  |> line(to = [0, baseHeight])
  |> close()

// Base solid
baseSolid = extrude(baseSketch, length = baseThickness)
  |> fillet(edges = all, radius = filletRadius)

// Hole sketch
holeSketch = startSketchOn(baseSolid, face = "top")
  |> circle(center = [holeOffset, holeOffset], radius = holeRadius)

// Cut the hole
finalSolid = subtract(baseSolid, extrude(holeSketch, length = baseThickness))

// Export
export(finalSolid, format = "stl", filename = "part.stl")
```

This is a complete parametric model. Change any parameter at the top, the whole model regenerates.

## What You Just Learned

1. The pipe operator transforms values through a sequence of functions
2. Sketch blocks track position and build paths incrementally
3. Tags let you name and reference geometry
4. Parametric design uses variables and dependencies
5. Constraints define geometric relationships (parallel, perpendicular, etc.)
6. 3D operations (extrude, revolve, sweep) create solids from sketches
7. Boolean operations combine solids (union, subtract, intersect)
8. Fillets and chamfers modify edges
9. Patterns replicate geometry
10. Transforms move, rotate, and scale geometry

## Exercises

1. **Build a parametric bracket**: Create a bracket with adjustable width, height, thickness, and hole positions. Add fillets to all edges.

2. **Use constraints**: Build a sketch with geometric constraints. Create a rectangle where opposite sides are parallel and adjacent sides are perpendicular. Fix one side's length and let the solver adjust the rest.

3. **Create a pattern**: Build a plate with a circular pattern of holes. Make the hole count and radius parametric.

4. **Implement a sweep**: Create a handrail by sweeping a circular profile along a curved path. Make the path radius and profile radius parametric.

5. **Complex boolean operations**: Build two overlapping shapes, subtract one from the other, then fillet the resulting edges. Experiment with different combinations.

## Next Up

In Part 9, we'll build a Jupyter kernel for KCL. You'll implement the kernel protocol, use KCL as a library, handle execution requests, and display geometry results in notebooks. This brings together everything you've learned: Rust, KCL internals, async programming, and the stdlib.

Time to build something real.
