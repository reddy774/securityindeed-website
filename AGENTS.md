# securityindeed-website â€” Agent Instructions

> **This is the canonical, LLM-agnostic project brain.** Any AI agent â€” Claude, GPT, Gemini, Copilot, Cursor, Windsurf, Continue, Aider â€” should read this file first. Tool-specific adapter files (`CLAUDE.md`, `.cursorrules`, `GEMINI.md`, `.github/copilot-instructions.md`) all point back to this document. Edit this file, not the adapters.

## What this project does

TODO: Write a 1-2 paragraph description of what this project is, who it serves, and why it exists.

## Current status

TODO: Is this production? Experimental? Archived? What's the last shipped version? What's broken right now?

## Tech stack

TODO: List the main technologies, frameworks, and tools. Include versions where they matter.

## How to run it locally

TODO: Exact commands for dev, test, build, deploy. Someone new should be able to follow these blind.

`ash
# Install dependencies
# TODO

# Start dev server
# TODO

# Run tests
# TODO
`

## Architecture

TODO: High-level design. Where does code live, how does data flow, what are the key abstractions?

## Conventions

TODO: Coding style, commit message format, branching strategy, review process.

## Pending tasks (priority order)

TODO: What's next? Replace this with real tasks from your backlog.

1. [ ] Task 1
2. [ ] Task 2

## LLM switching guide

This project uses the **canonical-adapter pattern** for LLM-agnosticism:

- **Canonical:** `AGENTS.md` (this file). Everything important lives here.
- **Adapters** point back here:
  - `CLAUDE.md` â€” `@AGENTS.md` import for Claude Code
  - `.cursorrules` â€” pointer for Cursor
  - `GEMINI.md` â€” pointer for Gemini CLI
  - `.github/copilot-instructions.md` â€” pointer for GitHub Copilot

When you switch LLMs, no knowledge is lost. Just open the project in the new tool; it reads its adapter, follows back here, and has full context.

---

*Bootstrapped via `dev/scripts/bootstrap-project.ps1` on 2026-04-12. Fill in the TODOs above.*
