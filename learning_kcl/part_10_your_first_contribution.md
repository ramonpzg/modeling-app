# Part 10: Your First Contribution to KCL

You've learned Rust. You've learned KCL. You've built a Jupyter kernel. Now you'll contribute it to the project. This part covers testing, documentation, code review, and the pull request process.

Open source contribution isn't just about code. It's about communication, collaboration, and making your work understandable to others. Let's do this right.

## Before You Code: Check the Project

Before writing anything, check if someone else is working on it:

1. **Search existing issues**: Go to the KCL repository and search issues for "jupyter" or "kernel". Someone might have already started this.

2. **Check pull requests**: Look at open PRs. Your feature might already be in progress.

3. **Ask in discussions**: If the project has discussions or a Discord/Slack, ask "I'm thinking of building a Jupyter kernel. Any interest? Anyone already working on this?"

If nobody's working on it, create an issue proposing the feature. Explain what you want to build, why it's useful, and how you plan to implement it. Get feedback before writing code.

Why? You might learn:
- The maintainers don't want this feature (rare, but possible)
- Someone tried before and hit a blocker you should know about
- There's a better approach you hadn't considered
- The maintainers will love it and give you guidance

In this case, a Jupyter kernel is clearly useful. Proceed.

## Project Structure Decisions

Where does the kernel live?

**Option 1: In the main repository** (`modeling-app/rust/kcl-jupyter/`)
- Pros: Official support, maintained with KCL
- Cons: Adds to the repository size, requires maintainer approval

**Option 2: Separate repository** (`your-username/kcl-jupyter`)
- Pros: You control it, faster iteration
- Cons: Less discoverable, might diverge from KCL

For a first contribution, separate is safer. You can always merge it into the main repo later if the maintainers want it.

But for this tutorial, let's assume it goes into the main repo.

## Writing Tests

Tests prove your code works and prevent regressions. KCL uses `cargo test` for Rust tests. Your kernel needs:

### Unit Tests

Test individual components:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_parse_and_execute() {
        let code = "width = 100\nheight = 50\narea = width * height";
        let mut state = KernelState::new();

        let result = state.execute(code).unwrap();

        assert_eq!(result.variables.get("width"), Some(&KclValue::Number(100.0, None)));
        assert_eq!(result.variables.get("height"), Some(&KclValue::Number(50.0, None)));
        assert_eq!(result.variables.get("area"), Some(&KclValue::Number(5000.0, None)));
    }

    #[test]
    fn test_error_handling() {
        let code = "x = undefined_var";
        let mut state = KernelState::new();

        let result = state.execute(code);

        assert!(result.is_err());
        let error = result.unwrap_err();
        assert!(error.message.contains("undefined"));
    }

    #[test]
    fn test_stateful_execution() {
        let mut state = KernelState::new();

        // First cell
        state.execute("x = 10").unwrap();

        // Second cell uses x
        let result = state.execute("y = x * 2").unwrap();

        assert_eq!(result.variables.get("y"), Some(&KclValue::Number(20.0, None)));
    }
}
```

Run them:

```bash
cargo test --package kcl-jupyter
```

### Integration Tests

Test the full kernel with real Jupyter:

```python
# test_kernel_integration.py
import pytest
from jupyter_client import KernelManager

@pytest.fixture
def kernel():
    """Start a KCL kernel for testing."""
    km = KernelManager(kernel_name='kcl')
    km.start_kernel()
    kc = km.client()
    kc.start_channels()
    kc.wait_for_ready()

    yield kc

    kc.stop_channels()
    km.shutdown_kernel()

def test_execute_simple_code(kernel):
    """Test executing simple KCL code."""
    msg_id = kernel.execute("width = 100")

    # Wait for execution to complete
    while True:
        msg = kernel.get_iopub_msg(timeout=10)
        if msg['msg_type'] == 'execute_reply':
            break

    assert msg['content']['status'] == 'ok'

def test_stateful_execution(kernel):
    """Test that variables persist between executions."""
    # First cell
    kernel.execute("x = 10")
    kernel.wait_for_idle()

    # Second cell
    msg_id = kernel.execute("y = x * 2")

    # Check result
    while True:
        msg = kernel.get_iopub_msg(timeout=10)
        if msg['msg_type'] == 'execute_result':
            assert 'y = 20' in str(msg['content']['data'])
            break

def test_error_handling(kernel):
    """Test error messages."""
    kernel.execute("bad = undefined_variable")

    while True:
        msg = kernel.get_iopub_msg(timeout=10)
        if msg['msg_type'] == 'error':
            assert 'undefined' in msg['content']['evalue'].lower()
            break
```

Run them:

```bash
pytest test_kernel_integration.py
```

## Documentation

Good documentation makes your contribution usable. Write:

### README.md

```markdown
# KCL Jupyter Kernel

A Jupyter kernel for KCL, Zoo's CAD programming language.

## Installation

```bash
pip install kcl-jupyter-kernel
```

## Usage

1. Start Jupyter:
   ```bash
   jupyter notebook
   ```

2. Create a new notebook and select "KCL" as the kernel.

3. Write KCL code:
   ```kcl
   width = 100
   height = 50

   sketch = startSketchOn("XY")
     |> startProfile(at = [0, 0])
     |> line(to = [width, 0])
     |> line(to = [width, height])
     |> line(to = [0, height])
     |> close()

   solid = extrude(sketch, length = 25)
   ```

4. Run the cell. The geometry appears below.

## Features

- Full KCL language support
- Stateful execution (variables persist between cells)
- 3D geometry visualization
- Autocomplete and documentation (Shift+Tab)
- Error messages with source context

## Development

Build from source:

```bash
git clone https://github.com/KittyCAD/modeling-app
cd modeling-app/rust/kcl-jupyter
cargo build --release
```

Run tests:

```bash
cargo test
pytest tests/
```

## License

MIT
```

### Code Comments

Don't over-comment. Good code is self-documenting. But explain non-obvious decisions:

```rust
// We clone the memory here because Jupyter cells can be re-executed
// out of order, and we don't want one cell's failure to corrupt
// the state for subsequent cells.
let memory_snapshot = self.memory.clone();
```

Avoid comments that repeat the code:

```rust
// BAD: This increments the counter
counter += 1;

// GOOD: Track execution count for Jupyter's execute_count field
self.execution_count += 1;
```

### API Documentation

Use rustdoc for public APIs:

```rust
/// Execute KCL code and return the result.
///
/// # Arguments
///
/// * `code` - The KCL source code to execute
///
/// # Returns
///
/// An `ExecutionResult` containing variables, stdout/stderr, and any geometry produced.
///
/// # Errors
///
/// Returns a `KclError` if parsing or execution fails.
///
/// # Examples
///
/// ```
/// let mut state = KernelState::new();
/// let result = state.execute("x = 10").unwrap();
/// assert_eq!(result.variables.get("x"), Some(&KclValue::Number(10.0, None)));
/// ```
pub async fn execute(&mut self, code: &str) -> Result<ExecutionResult, KclError> {
    // ...
}
```

Generate docs:

```bash
cargo doc --open
```

## Code Quality

Run the project's linters and formatters:

```bash
# Format code
cargo fmt

# Lint
cargo clippy -- -D warnings

# Check for common issues
cargo check
```

Fix any warnings. Clippy catches common mistakes:

```rust
// Clippy warns: this if/else can be simplified
if condition {
    true
} else {
    false
}

// Better:
condition
```

Follow the project's style. Look at existing code and match it.

## The Commit Message

Write clear commit messages:

```
Add Jupyter kernel for KCL

Implements a Jupyter kernel that allows running KCL code in notebooks.

Features:
- Stateful execution (variables persist between cells)
- Geometry visualization (exports to PNG/STL)
- Error handling with source context
- Autocomplete and documentation

This makes KCL accessible to users familiar with Jupyter notebooks
and enables interactive CAD programming workflows.

Closes #1234
```

The format:
1. **Subject line**: Summary in 50 characters or less, imperative mood ("Add X", not "Added X" or "Adds X")
2. **Body**: Explain what and why (not how, the code shows how)
3. **References**: Link to related issues ("Closes #1234", "Fixes #5678")

## Creating the Pull Request

Push your branch:

```bash
git checkout -b feature/jupyter-kernel
git add .
git commit -m "Add Jupyter kernel for KCL"
git push origin feature/jupyter-kernel
```

Go to GitHub and create a pull request. The description should include:

```markdown
## Summary

This PR adds a Jupyter kernel for KCL, enabling interactive CAD programming in notebooks.

## Changes

- New `kcl-jupyter` crate with kernel implementation
- Python bindings for kernel integration
- Geometry visualization (PNG export)
- Documentation and examples
- Integration tests

## Testing

Tested with Jupyter Notebook 6.4+ and JupyterLab 3.0+.

To test:
1. Install the kernel: `pip install -e ./rust/kcl-jupyter`
2. Start Jupyter: `jupyter notebook`
3. Run the example notebook: `examples/demo.ipynb`

## Checklist

- [x] Code follows project style
- [x] Tests added and passing
- [x] Documentation written
- [x] No clippy warnings
- [x] Formatted with `cargo fmt`

## Screenshots

[Screenshot of notebook with KCL code and rendered geometry]

## Related Issues

Closes #1234
```

Attach screenshots or GIFs showing the feature in action. Visuals help reviewers understand what you've built.

## The Code Review Process

Expect feedback. Good reviewers will:
- Suggest improvements
- Point out edge cases you missed
- Ask questions about your approach
- Request changes

Respond professionally:

**Good response:**
> Thanks for the feedback! I agree the error handling could be more robust. I'll add specific error types for parse errors vs execution errors and update the tests.

**Bad response:**
> This code works fine. I tested it and it's good enough.

If you disagree with feedback, explain your reasoning politely:

> I considered returning an Option here, but Result provides more context when things fail. The Jupyter protocol expects an error message, not just None. What do you think?

Make requested changes:

```bash
# Make changes
git add .
git commit -m "Address review feedback: improve error handling"
git push origin feature/jupyter-kernel
```

The PR updates automatically. Continue until reviewers approve.

## After the Merge

Your PR is merged. Congratulations! But you're not done:

1. **Monitor for issues**: Watch the repository. If someone reports a bug in your feature, fix it.

2. **Respond to questions**: Users might ask how to use your feature. Help them.

3. **Keep improving**: Add features, fix bugs, write more documentation.

4. **Help others**: Review other contributors' PRs. Share what you learned.

## Beyond the Jupyter Kernel

Now that you've contributed once, what's next?

### Small Improvements

- Add more stdlib functions
- Improve error messages
- Optimize performance
- Write more documentation

### Medium Features

- Language server improvements (better autocomplete)
- New constraint types
- Better geometry visualization

### Large Projects

- Implement a new backend (compile KCL to WASM)
- Add a macro system
- Build a debugger

### Maintenance

- Triage issues
- Review PRs
- Help new contributors
- Improve CI/CD

## The Bigger Picture

You started knowing nothing about Rust or KCL. Now you're a contributor. You understand:
- Rust's ownership, traits, generics, async, error handling
- KCL's parser, AST, executor, stdlib, type system
- CAD programming concepts
- Open source contribution process

This knowledge transfers. Rust skills apply to any Rust project. Compiler knowledge helps with other languages. CAD concepts work in any modeling system.

Keep learning. Keep building. Keep contributing.

## What You Just Learned

1. Check existing work before starting (issues, PRs, discussions)
2. Write tests (unit and integration)
3. Document code (README, comments, API docs)
4. Follow project style (formatting, linting)
5. Write clear commit messages
6. Create detailed pull requests with screenshots
7. Respond professionally to code review
8. Continue maintaining your contribution after merge

## Final Exercises

1. **Implement the Jupyter kernel**: Take what you learned in Part 9 and build a complete, tested, documented kernel.

2. **Create a pull request**: Open a PR to the KCL repository (or your own fork) with your kernel implementation.

3. **Add a stdlib function**: Pick a useful function that's missing from KCL's stdlib. Implement it, test it, document it, and contribute it.

4. **Improve documentation**: Find a poorly documented part of KCL. Write clear documentation and examples.

5. **Help another contributor**: Find an open PR that needs review. Read the code, test it, and provide constructive feedback.

## Where to Go From Here

### More Rust

- **The Rust Book**: Read the official book cover-to-cover
- **Rust by Example**: Hands-on examples of every Rust feature
- **Rustlings**: Small exercises to practice Rust concepts
- **Async Book**: Deep knowledge on async/await

### More CAD

- **Parametric modeling**: Learn tools like OpenSCAD, FreeCAD, Fusion 360
- **Computational geometry**: Study algorithms for mesh processing, CSG, constraints
- **Graphics programming**: Learn WebGL/WebGPU for rendering

### More Compilers

- **Crafting Interpreters**: Build a complete language from scratch
- **Engineering a Compiler**: Deep theory on parsing, optimization, code generation
- **LLVM**: Learn the compiler infrastructure KCL might use in the future

### More Open Source

- **Contribute widely**: Don't limit yourself to KCL. Contribute to other projects.
- **Start your own**: Build something new and invite contributors.
- **Teach others**: Write tutorials, give talks, mentor newcomers.

## The End (and the Beginning)

You started with a question: "How do I learn KCL and Rust to contribute?"

You now have the answer. You've gone from complete novice to contributor. You understand the language, the implementation, and the ecosystem.

But this is just the beginning. The real learning happens when you build things, break things, and fix things. When you read others' code and write code others will read. When you struggle with a bug for hours and finally fix it. When you help someone else learn what you now know.

KCL is a young language. Your contributions shape its future. The Jupyter kernel you built might become how thousands of people learn CAD programming. The stdlib functions you add might power real products. The documentation you write might be someone's first introduction to the language.

You have the skills. You have the knowledge. Now go build something remarkable.

Welcome to the KCL community. We're glad you're here.
