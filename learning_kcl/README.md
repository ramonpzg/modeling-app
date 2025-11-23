# Learning KCL: From Zero to Contributor

A comprehensive, hands-on tutorial series that teaches you Rust and KCL simultaneously, taking you from complete beginner to open source contributor.

## What This Is

This is a 10-part tutorial series designed to teach you:
- **Rust fundamentals** (ownership, traits, async, error handling)
- **KCL deeply** (parsing, execution, stdlib, CAD features)
- **Practical skills** to contribute to the KCL project

The goal: By the end, you'll have built a Jupyter kernel for KCL and be ready to contribute meaningfully to the language itself.

## Who This Is For

This series is designed for:
- **Advanced Python programmers** new to Rust and KCL
- **CAD enthusiasts** who want to learn programming for CAD
- **Open source contributors** looking to contribute to KCL
- **Language learners** interested in how programming languages work

You should be comfortable with:
- Programming concepts (variables, functions, loops, etc.)
- Python (comparisons are made throughout)
- Command line basics

You don't need to know:
- Rust (taught from scratch)
- KCL (taught from scratch)
- Compiler theory (explained as needed)
- CAD programming (introduced gradually)

## Teaching Philosophy

This series follows a **top-down, hands-on approach**:
- Code from the start (no 200 pages of theory first)
- Work backwards from the goal (start with running KCL, then understand how)
- Practical over theoretical (build real tools, not toy examples)
- Comparative learning (Python analogies throughout)

The tone is:
- Direct and terse (no marketing fluff)
- Witty with subtle sarcasm (to keep you engaged)
- Technical but accessible (complex ideas, simple words)
- Honest about complexity (Rust is hard, we won't pretend otherwise)

## The Journey

### Part 1: Your First KCL Program
**What you'll learn:** Rust basics, ownership, borrowing, running KCL code

Start by running KCL immediately, then learn just enough Rust to understand what happened. Compare Rust's ownership model to Python's reference counting. Understand why KCL is both a language and a runtime.

**Time investment:** 2-3 hours
**Prerequisites:** None

---

### Part 2: Ownership and the AST
**What you'll learn:** Deep ownership, AST structure, memory management

Build a tool that parses KCL and prints its AST. Understand why `Box<T>` is everywhere, how borrowing prevents bugs, and why Rust's approach makes large ASTs faster than Python's reference counting.

**Time investment:** 3-4 hours
**Prerequisites:** Part 1

---

### Part 3: Pattern Matching and Parsing
**What you'll learn:** Enums, pattern matching, tokenizing, parser combinators

Build a tokenizer and simple parser. Understand how KCL transforms source code into an AST using recursive descent parsing and the winnow library. Learn why Rust's enums are nothing like Python's Enum.

**Time investment:** 4-5 hours
**Prerequisites:** Parts 1-2

---

### Part 4: Traits, Generics, and Types
**What you'll learn:** Traits, generics, KCL's runtime type system

Understand how Rust's compile-time types differ from KCL's runtime types. Build a simple type checker. Learn traits (Rust's interfaces), generics (code for any type), and how KCL uses them throughout.

**Time investment:** 3-4 hours
**Prerequisites:** Parts 1-3

---

### Part 5: Error Handling
**What you'll learn:** Result, Option, error types, error reporting

Learn Rust's error handling without exceptions. Build an error reporter that shows source context like a real compiler. Understand when to use `Result`, `Option`, and `panic!`.

**Time investment:** 2-3 hours
**Prerequisites:** Parts 1-4

---

### Part 6: Async and Execution
**What you'll learn:** Async/await, futures, KCL's executor, engine communication

Understand how KCL talks to the geometry engine asynchronously. Learn Rust's zero-cost async (very different from Python's async). Build an async evaluator that executes KCL code.

**Time investment:** 4-5 hours
**Prerequisites:** Parts 1-5

---

### Part 7: Standard Library Deep Dive
**What you'll learn:** Stdlib architecture, implementing functions, registration

Explore KCL's two-layer stdlib (KCL + Rust). Implement your own stdlib function. Understand argument parsing, units, and how functions integrate with the geometry engine.

**Time investment:** 3-4 hours
**Prerequisites:** Parts 1-6

---

### Part 8: Advanced KCL Features
**What you'll learn:** Pipes, sketches, constraints, parametric design

Learn what makes KCL a CAD language. Understand pipe operators, sketch blocks, tags, parametric relationships, and geometric constraints. Build complex 3D models with code.

**Time investment:** 3-4 hours
**Prerequisites:** Parts 1-7

---

### Part 9: Building a Jupyter Kernel
**What you'll learn:** Jupyter protocol, kernel implementation, geometry display

Build a Jupyter kernel for KCL. Implement stateful execution, geometry visualization, autocomplete, and error reporting. Choose between Python (simpler) or Rust (faster) implementation.

**Time investment:** 6-8 hours
**Prerequisites:** Parts 1-8

---

### Part 10: Your First Contribution
**What you'll learn:** Testing, documentation, code review, pull requests

Polish your Jupyter kernel for contribution. Write tests, documentation, and create a pull request. Learn the open source contribution process and what comes after your first merge.

**Time investment:** 4-6 hours
**Prerequisites:** Parts 1-9

---

## Total Time Investment

**40-50 hours** spread over several weeks. This assumes:
- You do the reading (not just skimming)
- You complete the exercises (critical for learning)
- You experiment beyond the examples (encouraged)

This is not a weekend project. It's a serious investment in learning two non-trivial technologies deeply.

## How to Use This Series

### Sequential Learning (Recommended)
Work through parts in order. Each builds on previous knowledge.

### Reference Learning
Already know Rust? Skip Parts 1-6, start at Part 7.
Already know KCL? Focus on Rust parts (1-6), skim KCL parts.
Just want the Jupyter kernel? Read Part 9, backtrack as needed.

### Study Group Learning
Work through this with others. Discuss exercises. Compare implementations.

## What You'll Build

By the end, you'll have:
- A **KCL AST printer** (Part 2)
- A **mini parser** for arithmetic (Part 3)
- A **type checker** (Part 4)
- An **error reporter** with source context (Part 5)
- An **async evaluator** (Part 6)
- Your **own stdlib function** (Part 7)
- **Parametric CAD models** (Part 8)
- A **complete Jupyter kernel** (Part 9)
- A **pull request** to the KCL project (Part 10)

## Beyond This Series

After completing this, you'll be ready for:
- Contributing to KCL's parser, executor, or stdlib
- Building your own DSL in Rust
- Understanding other Rust projects (compilers, databases, web frameworks)
- Teaching others Rust and KCL

## Getting Help

Stuck? Resources:
- **KCL Discord/Slack**: Ask the community
- **Rust forums**: https://users.rust-lang.org/
- **Stack Overflow**: Tag questions with `rust` and `kcl`
- **The Rust Book**: https://doc.rust-lang.org/book/
- **KCL documentation**: Check the official docs

When asking for help:
1. Show what you've tried
2. Include error messages
3. Provide a minimal example
4. Explain what you expected vs what happened

## Contributing to This Series

Found a mistake? Have a suggestion? Want to add content?

This series itself is open to contributions. Submit issues or PRs to improve it.

## Acknowledgments

This series was created to teach Rust and KCL in the style of:
- **Fast.ai** (Jeremy Howard's top-down approach)
- **Crafting Interpreters** (Bob Nystrom's clarity)
- **The Rust Book** (thoroughness without condescension)
- **Learn You a Haskell** (wit and personality)

Special thanks to the KCL maintainers for building a language worth learning.

## License

This tutorial series is licensed under MIT. Use it, modify it, share it.

The code examples are in the public domain. Use them however you want.

## Ready to Start?

Open [Part 1: Your First KCL Program](part_01_your_first_kcl_program.md) and begin your journey.

Remember: Everyone who knows Rust was once where you are now. The difference between them and you is they started. You're about to.

Good luck. You'll need it. But you'll also succeed if you persist.
