# CLAUDE.md - Instructions for AI Assistants

## Project Overview

`run-command-safely` is a security wrapper script that acts as a gateway between AI agents (Cursor, Claude Code, Aider, etc.) and the underlying system. It enforces granular permissions on which commands can be run, with what arguments, and in which directories.

## Key Commands

```bash
# Run tests (when implemented)
./test.sh

# Run the main script
./run-command-safely.sh [OPTIONS] COMMAND [ARGS...]

# Check if a command would be allowed
./run-command-safely.sh --dry-run COMMAND [ARGS...]

# List allowed commands
./run-command-safely.sh --allowed-commands
```

## Configuration Location

- Default: `~/.config/run-command-safely/config.yaml`
- Override: `RUN_COMMAND_SAFELY_CONFIG` env var or `--config` flag

## Reserved Configuration Keys

When working with the YAML configuration:
- `allowed-dirs` - Directory restrictions (glob patterns)
- `allowed-args` - Argument whitelist
- `description` - Help text
- `denied` - Explicit denial with reason
- `blocked` - Block entire command with reason

All other keys are interpreted as subcommands.

---

## Document Purposes

Each root markdown file has a specific purpose. Put content in the right place:

| File | Purpose | Content Examples |
|------|---------|------------------|
| **README.md** | Brief project overview | What it does, how to install, how to run |
| **CLAUDE.md** | Instructions for AI assistants | Workflow, gotchas, environment notes |
| **ARCHITECTURE.md** | System structure | Components, data models, data flow |
| **DESIGN.md** | Decisions and reasoning | Why choices were made, tradeoffs considered |
| **TODO.md** | Future work ideas | Things that came up, shouldn't be forgotten |

**Rule of thumb:**
- "What is this project?" → README
- "How do I work in this repo?" → CLAUDE
- "How does the system work?" → ARCHITECTURE
- "Why was it built this way?" → DESIGN
- "What might we do later?" → TODO

### What Goes in ARCHITECTURE.md

**Include:**
- Component responsibilities and boundaries
- Data flow between components
- How to instantiate and use components
- Behavior that affects multiple components or callers
- Error handling philosophy (what propagates vs what's handled)

**Leave in code:**
- Internal implementation details (caching, sentinel values, internal helpers)
- Details only relevant when modifying that specific file

**Test:** Would a new agent working on a *different* component benefit from knowing this? If yes, document it. If only useful when reading *this* file, leave it in code.

---

## CRITICAL: Keep Docs in Sync

**This is a CRITICAL severity requirement. Outdated documentation creates wrong code.**

Each agent session starts with fresh context. Agents read ARCHITECTURE.md and DESIGN.md to understand the system before making changes. If docs don't match implementation:

1. The next agent will write code that doesn't work
2. The agent will trust the docs and not verify against actual implementation
3. Wrong assumptions compound into architectural drift

**WHY and INTENT matter more than WHAT:**

The next agent can read code to see what it does. What they cannot see is WHY it was designed that way. Without the reasoning, agents make "fixes" that break things they didn't understand.

**Where to put WHY:**
- **Inline comments** — Local implementation choices affecting one file
- **DESIGN.md** — Significant decisions affecting multiple files or system architecture

---

## Writing Code for LLM Agents

LLM coding agents experience code more like grep output, not like a human using an IDE. LLMs see what is in their context-window, not the full project. Write code accordingly.

**1. Optimize for Grep, Not for IDE**

Names should be unique enough to locate via text search without false positives. Prefer `get_user_by_email()` over `get()`. Prefer `payment_stripe.py` over `utils.py`.

**2. One Concept Per File, Named for What It Contains**

Each file should contain one cohesive concept. If you struggle to name a file, it probably contains too many things.

**3. Inline Over Fragmented**

Prefer code that reads linearly from top to bottom over code that jumps between many small helper methods. Extract only when there's clear reuse or a genuine abstraction.

**4. Explicit Over Implicit**

Write code as if the reader cannot easily access files except the one they're viewing. Avoid decorators, metaprogramming, and "magic" that hides behavior.

**5. Types as Documentation**

Annotate and use types wherever possible as if they may be the only documentation available.

**6. Errors That Explain Themselves**

Write error messages that state what went wrong, relevant variable context, and what was expected.

**7. Comments That State Intent**

Comments should explain why code exists and what it expects, not just what it does. Write comments like a colleague explaining context you'd need before modifying something.

**8. Colocate What Changes Together**

Related code that changes together should live together, even if architectural patterns suggest separating by type.

**9. Tests as Executable Specifications**

Write tests in the style of executable documentation. An agent who reads only your test file should understand the complete contract of the code under test.

---

## Environment

**Git:** The `main` branch exists only on the remote, not locally.

```bash
git diff origin/main..HEAD   # Correct
git diff main..HEAD          # Wrong - fails with "unknown revision"
```

---

## Design Decision Format

When adding to DESIGN.md:

```markdown
# Descriptive Name

Keywords: searchable terms for grep
Date: YYYYMMDD

Paragraph explaining the problem, the decision, and WHY.
```
