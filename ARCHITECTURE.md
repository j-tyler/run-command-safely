# Architecture

## System Overview

`run-command-safely` is a shell script that intercepts commands from AI agents and validates them against a YAML configuration before execution.

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│    AI Agent     │────▶│  run-command-safely  │────▶│  System Shell   │
│ (Cursor, Claude)│     │       (gateway)      │     │   (if allowed)  │
└─────────────────┘     └──────────────────────┘     └─────────────────┘
                                  │
                                  ▼
                        ┌──────────────────────┐
                        │    config.yaml       │
                        │  (permission rules)  │
                        └──────────────────────┘
```

## Core Components

### 1. Configuration Parser

Reads and parses YAML configuration file into a traversable tree structure.

**Responsibilities:**
- Load config from default location or specified path
- Validate configuration structure
- Build command tree for efficient matching

**Location:** Main script, config parsing section

### 2. Command Matcher

Implements the tree traversal algorithm to determine if a command is allowed.

**Responsibilities:**
- Match command against config tree
- Handle subcommand hierarchy
- Validate arguments against whitelist
- Check directory restrictions

**Location:** Main script, matching section

### 3. Directory Validator

Checks if current working directory matches allowed patterns.

**Responsibilities:**
- Expand glob patterns
- Match CWD against allowed directories
- Handle inheritance of directory restrictions

**Location:** Main script, directory validation section

### 4. Error Reporter

Generates LLM-friendly error messages.

**Responsibilities:**
- Format denial reasons clearly
- Suggest allowed alternatives
- Provide actionable feedback

**Location:** Main script, output section

## Data Flow

### Command Validation Flow

```
1. Parse command line arguments
2. Load configuration file
3. Look up base command in config
4. If not found → DENY (unknown command)
5. If blocked → DENY (blocked)
6. Check directory restrictions → DENY if not allowed
7. Traverse argument tree:
   a. For each argument, check if it's a subcommand
   b. If subcommand has 'denied' → DENY
   c. If subcommand has directory override → check
   d. Continue traversal
   e. If no subcommand match, check allowed-args
   f. If arg not in allowed-args → DENY
8. All checks passed → ALLOW and execute
```

### Configuration Inheritance

Directory restrictions (`allowed-dirs`) cascade down the tree:

```yaml
git:
  allowed-dirs: [~/projects/**]  # Applies to all git subcommands
  log:                           # Inherits ~/projects/**
  stash:
    allowed-dirs: [~/trusted/**] # Overrides parent
    list:                        # Uses ~/trusted/**
```

## Key Design Decisions

### 1. Tree-Based Configuration

Commands and subcommands form a natural tree. The config mirrors this:

```yaml
git:
  stash:
    list:    # git stash list
    show:    # git stash show
```

### 2. Implicit Deny

Commands not in config are denied. This is safer than implicit allow.

### 3. Explicit Deny with Reasons

The `denied` key allows custom error messages that teach the AI agent:

```yaml
git:
  commit:
    denied: Commits not allowed in review mode
```

### 4. Directory Scoping

Commands can be restricted to specific directories, preventing access to sensitive areas.

### 5. LLM-Friendly Output

Error messages are designed to help AI agents understand:
- What was denied
- Why it was denied
- What alternatives are available

## Dependencies

- **bash** (4.0+) or POSIX shell
- **yq** or similar for YAML parsing (or built-in parsing)
- Standard Unix utilities (grep, sed, etc.)

## Security Considerations

1. **Config file permissions** - Should be readable only by user
2. **Path traversal** - Sanitize directory patterns
3. **Argument injection** - Validate all inputs
4. **Symlink handling** - Resolve symlinks in directory checks
5. **Shell metacharacters** - Properly quote all arguments
