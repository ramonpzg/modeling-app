# Part 7: From AST to Actual Stuff: The KCL Engine (Expanded)

We've followed the journey of our KCL code from raw text to a structured Abstract Syntax Tree. Now, we arrive at the destination: the engine. The engine, or more accurately, the "executor," is the part of the KCL compiler that takes the AST and brings it to life. If the parser is the architect drawing up the blueprints, the engine is the construction crew that builds the thing.

The code for the executor can be found in `rust/kcl-lib/src/execution`. The engine-specific communication logic is in `rust/kcl-lib/src/engine`.

## The Core Concept: Walking the Tree

The engine's primary job is to "walk" the AST. It traverses the tree, node by node, and for each node, it performs an action. This is a classic interpreter pattern.

-   If it sees a `VariableDeclaration` node, it evaluates the right-hand side expression and stores the result in memory.
-   If it sees a `CallExpression` node, it looks up the function being called, evaluates the arguments, and then executes the function.
-   If it sees a binary operation like `+`, it evaluates the left and right sides and then performs the addition.

This process is recursive. To evaluate a `VariableDeclaration`, you must first evaluate its expression. To evaluate a `CallExpression`, you must first evaluate its arguments, which might themselves be other expressions or function calls.

## The Execution Context: Memory and State

Where do variables and function definitions live? They're stored in an **Execution Context** (or "environment"). This is a data structure that holds the state of the program at any given point. It contains a "memory" component, which is essentially a map from variable names to their values.

When the engine encounters a `VariableDeclaration`, it adds a new entry to this map. When it encounters an identifier (a variable name), it looks it up in the map to get its value. This is how information flows through a KCL program.

## The Bridge to Geometry: The `EngineManager`

The KCL engine itself doesn't know how to draw a line or a circle. It knows how to manage variables and execute functions, but the actual geometry creation is handled by a separate, powerful geometry kernel.

The communication between the KCL engine and the geometry kernel is managed by the `EngineManager` trait, which you can see in `rust/kcl-lib/src/engine/mod.rs`. This trait defines the interface for sending commands to the geometry kernel.

When a standard library function like `circle` is called in KCL, its Rust implementation (`shapes::circle` in `kcl-lib`) is executed. This Rust function doesn't draw a circle directly. Instead, it constructs a `ModelingCmd`—a data structure that represents a command for the geometry kernel—and sends it via the `EngineManager`.

### Why Batch Commands?

You'll notice that the `EngineManager` has methods like `batch_modeling_cmd`. It doesn't just send one command at a time. Instead, it collects, or "batches," a series of commands together and sends them all at once.

This is for performance. Sending messages between different parts of a system (especially if they're running in different processes or over a network) has overhead. It's much more efficient to send one large message with 100 commands than to send 100 small messages, each with one command. The KCL engine builds up a batch of geometry commands as it executes your code, and then flushes this batch when it needs to.

### Let's Trace Our Example:

```kcl
const wheelRadius = 10
const wheel = circle(0, 0, wheelRadius)
```

1.  **AST Arrives**: The engine receives the AST for this program.
2.  **First Declaration**: It visits the first `VariableDeclaration` node for `wheelRadius`.
    -   It evaluates the value, which is the literal `10`.
    -   It stores `wheelRadius -> 10` in its memory.
3.  **Second Declaration**: It visits the second `VariableDeclaration` for `wheel`.
    -   It needs to evaluate the `CallExpression` `circle(...)`.
    -   It evaluates the arguments: the literals `0`, `0`, and the identifier `wheelRadius`. It looks up `wheelRadius` in memory and finds `10`.
    -   It now knows it needs to call the `circle` function with arguments `(0, 0, 10)`.
4.  **Function Execution**: It calls the Rust function `shapes::circle`.
    -   Inside `shapes::circle`, a `ModelingCmd::MakeCircle` command is created.
    -   This command is passed to the `EngineManager`'s `batch_modeling_cmd` method and added to a queue.
5.  **Return and Store**: The `shapes::circle` function returns a `KclValue` representing the sketch that was created.
    -   The engine stores this value in its memory: `wheel -> <SketchObject>`.
6.  **End of Program**: The program is done. The engine might then flush the batch of commands, sending the `MakeCircle` command to the geometry kernel, which then actually creates the circle.

This is the core loop of the KCL engine: evaluate, store, and dispatch commands. It's a beautiful dance between the interpreted world of KCL and the compiled, high-performance world of the geometry kernel.

Now that you understand how the engine brings your code to life, we're ready for the final act: building a Jupyter kernel. This will tie together everything we've learned, as we'll be building our own client that interacts with the KCL engine.
