# Skills

Collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) custom skills.

## Available Skills

### `nodejs-performance-expert`

Expert-level Node.js performance skill covering V8 internals (heap, GC, JIT) and libuv architecture (event loop phases, thread pool, async I/O). Activates for writing, reviewing, optimizing, or debugging Node.js code.

Covers:
- V8 memory model & garbage collection
- JIT compilation pipeline & optimization rules
- Event loop phases & blocking detection
- Thread pool sizing & saturation
- Production configuration & diagnostics

## Structure

```
skills/
├── nodejs-performance-expert/
│   ├── SKILL.md              # Skill definition
│   └── references/           # Deep-dive reference docs
│       ├── v8-gc-deep.md
│       ├── event-loop-phases.md
│       └── thread-pool-ops.md
└── README.md
```
