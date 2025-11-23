# Part 9: The Nitty-Gritty: Implementing the Kernel (Expanded)

We have our skeleton. Now it's time to give it a soul. In this part, we will implement the `execute` method of our `KclKernel`. This is the core of our project, where we take KCL code from the user, pass it to the `kcl-lib` executor, and then translate the result into a format that Jupyter can understand and display.

## The `execute` Method: From Code to Content

Let's revisit the `execute` method in our `src/main.rs`. Its job is to take a string of code and return a `Result` containing a vector of `DisplayData`. `DisplayData` is a struct from the `jupyter_kernel` crate that represents a rich output, which can have multiple formats (like plain text, HTML, images, etc.).

Our strategy will be:
1.  Acquire a lock on our shared `Executor` instance.
2.  Clear the executor's previous state.
3.  Execute the new code.
4.  If it's an error, format it for Jupyter.
5.  If it's a success, export the resulting geometry as a GLTF file.
6.  Encode the GLTF data as Base64.
7.  Create an HTML `DisplayData` object that uses a JavaScript 3D viewer (`<model-viewer>`) to display the model.
8.  Return the `DisplayData`.

## The Implementation

Here's the new, expanded `execute` method. It's a lot to take in, but I'll break it down.

```rust
// In src/main.rs, add the missing `use` statement at the top.
use base64;

// ... inside the `impl Kernel for KclKernel` block.

async fn execute(&self, code: &str, execution_count: u32) -> Result<Vec<DisplayData>> {
    // Acquire a lock on the executor.
    // This gives us mutable access.
    let mut executor = self.executor.lock().await;

    // Clear the previous state.
    executor.clear().await?;

    // Execute the KCL code.
    let execution_result = executor.execute(code).await;

    match execution_result {
        Ok(_) => {
            // The execution was successful. Now, let's export the geometry.
            let gltf_data = executor.export_gltf().await?;

            // Encode the GLTF binary data as Base64.
            let base64_gltf = base64::encode(&gltf_data);

            // Create the HTML to display the model. We'll use Google's <model-viewer>.
            let html = format!(
                r#"
                <script type="module" src="https://ajax.googleapis.com/ajax/libs/model-viewer/3.0.1/model-viewer.min.js"></script>
                <model-viewer style="width: 100%; height: 400px;" src="data:model/gltf-binary;base64,{}" ar camera-controls></model-viewer>
                "#,
                base64_gltf
            );

            // Create a DisplayData object with the HTML.
            let display_data = DisplayData {
                data: json!({
                    "text/html": html,
                    "text/plain": "3D model successfully generated."
                }),
                ..Default::default()
            };

            Ok(vec![display_data])
        }
        Err(e) => {
            // There was an error during execution.
            let error_message = format!("KCL Execution Error:\n{}", e);
            let display_data = DisplayData {
                data: json!({
                    "text/plain": error_message
                }),
                ..Default::default()
            };
            Ok(vec![display_data])
        }
    }
}
```

We'll also need a new dependency for Base64 encoding. Add this to your `Cargo.toml` if you haven't already:

```toml
[dependencies]
# ... other dependencies ...
base64 = "0.21"
```

### Breaking It Down: The Right Way to Handle State

This is the most important lesson in this section. Notice that our `execute` method signature is `&self`, not `&mut self`. This is because we are implementing a trait, and the `jupyter_kernel::Kernel` trait defines `execute` with an immutable `&self` reference. We *cannot* change the signature.

So how do we modify the executor if we only have an immutable reference? This is a classic concurrency problem. The `jupyter_kernel` crate might call `execute` or other methods on our `Kernel` from different threads or asynchronous tasks. If we used `&mut self`, we could have data races.

The solution is **interior mutability**. We wrap our `Executor` in two layers:

1.  `tokio::sync::Mutex`: A `Mutex` (mutual exclusion) is a lock. It ensures that only one piece of code can access the data inside it at any given time. Before we can use the `Executor`, we must call `.lock().await`. This will wait until no one else is using the executor, and then it will give us a "lock guard" that provides mutable access. When the guard goes out of scope, the lock is released.
2.  `std::sync::Arc`: An `Arc` (Atomically Reference Counted) is a smart pointer that lets us have multiple "owners" of the same data. It keeps track of how many references to the data exist, and when the last reference is dropped, the data is cleaned up. This is necessary because the `jupyter_kernel` library will need to share our `KclKernel` instance across different parts of its async runtime.

By using `Arc<Mutex<Executor>>`, we can safely share our executor and modify it, even from a method that only has `&self`. This is a fundamental and crucial pattern for writing safe concurrent Rust.

The rest of the logic is as we discussed before: exporting the geometry to GLTF, encoding it, and using a `<model-viewer>` web component to display it in a rich HTML output.

## The Next Step: Installation and Testing

We now have a *correctly implemented* and functional KCL kernel. You can build it with `cargo build`. The next step is to make Jupyter aware of it. This involves creating a "kernelspec" file—a JSON file that tells Jupyter how to start our kernel—and placing it in a specific directory.

In the final part of our series, we'll cover how to write this kernelspec, how to install it, and how to finally run your KCL code in a real Jupyter notebook. We'll also discuss how to package this up for others to use, and how to make your first real contribution.
