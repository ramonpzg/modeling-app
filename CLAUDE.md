# Claude Agent Context: KCL Learning Project

## Project Overview

This repository is **modeling-app** by Zoo.dev, which contains KCL (KittyCAD Language) - a domain-specific language for parametric CAD programming.

The user (ramonpzg) is working through a comprehensive learning journey to:
1. **Learn Rust from scratch** (coming from advanced Python background)
2. **Learn KCL deeply** (parsing, execution, stdlib, CAD features)
3. **Contribute to the KCL project** with meaningful additions
4. **Build novel integrations** (Jupyter kernel, tldraw integration, AI-powered features)

## User Background

- **Strong Python programmer** (advanced level)
- **New to Rust** (learning from scratch)
- **New to KCL** (learning from scratch)
- **Interest in CAD programming** and parametric design
- **Goal**: Become a confident contributor to the KCL project

## Current Work

### Main Branch
- Working on: `claude/learn-kcl-language-019nRqq5g4vogDXQvKjRdmAg`
- Base branch: `main`

### Active Directory
- `learning_kcl/` - Comprehensive 13-part tutorial series (created by Claude)

## User's Goals

### Primary Goals
1. **Deep understanding of KCL** - parser, AST, executor, stdlib, type system
2. **Deep understanding of Rust** - ownership, traits, async, error handling
3. **Build a Jupyter kernel for KCL** - first major contribution
4. **Integrate KCL with tldraw** - novel infinite canvas CAD interface
5. **Implement AI-powered sketch-to-code** - makereal-style conversion for CAD

### Learning Style
- **Top-down, hands-on** - code from the start, understand theory as needed
- **Practical over theoretical** - build real tools, not toy examples
- **Comparative learning** - Python analogies help bridge understanding
- **Progressive complexity** - start simple, build toward advanced features

## Tutorial Series Structure

The user has a **13-part tutorial series** in `learning_kcl/`:

1. **Part 1**: Rust basics and first KCL program
2. **Part 2**: Ownership, borrowing, and KCL's AST
3. **Part 3**: Pattern matching, enums, and parsing
4. **Part 4**: Traits, generics, and KCL's type system
5. **Part 5**: Error handling without exceptions
6. **Part 6**: Async Rust and KCL's execution model
7. **Part 7**: Standard library architecture
8. **Part 8**: Advanced KCL features (sketches, pipes, parametric design)
9. **Part 9**: Building a Jupyter kernel for KCL
10. **Part 10**: First contribution (testing, docs, PR process)
11. **Part 11**: Integrating KCL with tldraw (infinite canvas)
12. **Part 12**: 3D geometry visualization in tldraw shapes
13. **Part 13**: AI-powered sketch-to-code conversion

## Key Project Features

### KCL Architecture
- **Location**: `rust/kcl-lib/` - main language implementation
- **Parser**: Uses winnow library, recursive descent
- **Executor**: Async execution, communicates with geometry engine
- **Stdlib**: Dual-layer (KCL + Rust implementations)
- **WASM**: Browser execution via `rust/kcl-wasm-lib/`

### Novel Integrations (User's Innovations)
1. **Jupyter Kernel**: Run KCL in notebooks with geometry visualization
2. **tldraw Integration**: Code blocks + 3D viewers on infinite canvas
3. **AI Conversion**: Hand-drawn sketches → parametric KCL code (GPT-4V/Claude)

## Important Context

### Technology Stack
- **Language**: Rust (backend), TypeScript/React (frontend)
- **CAD Engine**: Geometry engine via WebSocket/WASM
- **Canvas**: tldraw (React-based infinite canvas)
- **3D Rendering**: Three.js / React Three Fiber
- **AI**: OpenAI GPT-4V or Anthropic Claude (vision models)

### Code Style Preferences
- **Direct and terse** - no marketing fluff
- **Practical examples** - working code over theory
- **Python comparisons** - help bridge understanding
- **Wit and subtle sarcasm** - keep it engaging

### File Organization
- **Tutorials**: `learning_kcl/*.md`
- **KCL Source**: `rust/kcl-lib/src/`
- **Frontend**: `src/` (React/TypeScript)
- **Examples**: `public/kcl-samples/`

## Working with This User

### When Providing Help
1. **Check PROGRESS.md first** - see what's already been done
2. **Reference tutorial parts** - the user has comprehensive guides
3. **Provide working code** - practical over theoretical
4. **Python analogies** - when explaining Rust concepts
5. **Be direct** - no fluff, no excessive praise

### When Writing Code
- **Follow existing patterns** in the codebase
- **Add comments only where non-obvious**
- **Use descriptive variable names**
- **Prefer simplicity** - don't over-engineer

### When Explaining Concepts
- **Start with the end goal** - top-down approach
- **Show working examples** first
- **Explain theory** as needed to understand the practice
- **Compare to Python** when helpful

## Common Commands

### Building
```bash
cargo build --release
cargo test
```

### Running
```bash
npm run dev  # Frontend
cargo run --bin <binary>
```

### Linting
```bash
cargo fmt
cargo clippy -- -D warnings
```

### Git Workflow
```bash
# Currently on: claude/learn-kcl-language-019nRqq5g4vogDXQvKjRdmAg
git status
git add <files>
git commit -m "message"
git push -u origin claude/learn-kcl-language-019nRqq5g4vogDXQvKjRdmAg
```

## Important Notes

### Git Branch Naming
- Branch MUST start with `claude/` and end with session ID
- Current: `claude/learn-kcl-language-019nRqq5g4vogDXQvKjRdmAg`
- Push will fail with 403 if branch name is incorrect

### API Keys
- OpenAI or Anthropic API key needed for Part 13 (AI integration)
- Should be handled server-side, not client-side in production

### WASM Limitations
- KCL WASM bindings may not expose all functionality
- May need to extend bindings for new features

## Progress Tracking

**See PROGRESS.md for detailed session-by-session progress.**

That file contains:
- What has been built
- What was learned
- Current status of each tutorial
- Next steps and recommendations

## Questions for New Agents

When starting a new session, ask yourself:
1. Where is the user in the tutorial series?
2. What are they currently working on?
3. Have they hit any blockers?
4. What's the next logical step?

Check PROGRESS.md for answers to these questions.

## Resources

- **KCL Documentation**: Check the official docs
- **Rust Book**: https://doc.rust-lang.org/book/
- **tldraw Docs**: https://tldraw.dev/docs
- **Three.js Docs**: https://threejs.org/docs/
- **Tutorials**: All in `learning_kcl/`

## Contact

If unsure about anything:
1. Check PROGRESS.md
2. Read the relevant tutorial part
3. Look at existing code in the codebase
4. Ask the user for clarification

The user appreciates direct questions over assumptions.

---

*This file is maintained to provide context for Claude agents working with this user. Update PROGRESS.md after each session to keep context current.*
