## Implementation Plan: `run-command-safely.sh`

### A Safety Wrapper for Running Shell Commands with AI Agents

---

## 1. Executive Summary

### 1.1 Purpose

Create a shell script that acts as a security gateway between AI agents (Cursor, Claude Code, Aider, etc.) and the underlying system. The script enforces granular permissions on which commands can be run, with what arguments, and in which directories.

### 1.2 Key Features

- **Tree-based configuration** mirroring natural command hierarchy
- **Hierarchical matching** of command → subcommand → arguments
- **Directory scoping** to restrict commands to specific folders
- **LLM-friendly error messages** that teach correct usage
- **Agent-agnostic design** works with any AI coding assistant

### 1.3 Usage Examples

```bash
# Allowed - git log in trusted directory
$ run-command-safely.sh git log --oneline -10
<output of git log>

# Denied - git commit not allowed
$ run-command-safely.sh git commit -m "test"
✗ DENIED: git commit
REASON: Commits not allowed in review mode
...

# Help and discovery
$ run-command-safely.sh --allowed-commands
$ run-command-safely.sh --help git
$ run-command-safely.sh --help git branch
```

---

## 2. Configuration Format

### 2.1 File Location

```
~/.config/run-command-safely/config.yaml
```

Optional override via environment variable or flag:
```bash
RUN_COMMAND_SAFELY_CONFIG=/path/to/config.yaml
run-command-safely.sh --config /path/to/config.yaml
```

### 2.2 Configuration Structure

The configuration is a YAML file with a tree structure where:
- **Top-level keys** are command names
- **Nested keys** are subcommands or arguments
- **Reserved keys** (prefixed or specially named) provide metadata

#### 2.2.1 Reserved Keys

| Key | Scope | Type | Description |
|-----|-------|------|-------------|
| `allowed-dirs` | Any node | `List[str]` | Glob patterns for allowed directories. Inherited by children unless overridden. |
| `allowed-args` | Any node | `List[str]` | Whitelist of allowed arguments. If present, only these args are permitted. |
| `description` | Any node | `str` | Human-readable description for help output. |
| `denied` | Any node | `str` | Explicitly marks this path as denied. Value is the reason shown in error. |
| `blocked` | Root command only | `str` | Marks entire command as blocked. Value is the reason. |

#### 2.2.2 Interpretation Rules

1. **Presence without value** = allowed with any arguments
   ```yaml
   git:
     log:        # git log is allowed with any arguments
     diff:       # git diff is allowed with any arguments
   ```

2. **Presence with `allowed-args`** = only those arguments permitted
   ```yaml
   git:
     branch:
       allowed-args: [-l, -a, -v, --list]
   ```

3. **Nested keys** = subcommand hierarchy
   ```yaml
   git:
     stash:
       list:     # git stash list is allowed
       # git stash pop, git stash drop, etc. are implicitly denied
   ```

4. **`denied` key** = explicitly forbidden with custom message
   ```yaml
   git:
     commit:
       denied: Commits not allowed in review mode
   ```

5. **`blocked` key** = entire command forbidden
   ```yaml
   rm:
     blocked: Deleting files is not permitted
   ```

6. **`allowed-dirs` inheritance** = flows down to children
   ```yaml
   git:
     allowed-dirs:
       - ~/projects/trusted/**
     log:        # Inherits ~/projects/trusted/**
     diff:       # Inherits ~/projects/trusted/**
     stash:
       allowed-dirs:
         - ~/projects/extra-trusted/**  # Overrides parent
       list:     # Uses ~/projects/extra-trusted/**
   ```

### 2.3 Complete Example Configuration

```yaml
# ~/.config/run-command-safely/config.yaml
#
# Configuration for run-command-safely.sh
# Tree structure mirrors command hierarchy.
#
# RESERVED KEYS:
#   allowed-dirs   - List of glob patterns for directory restrictions
#   allowed-args   - List of allowed arguments (whitelist)
#   description    - Human-readable help text
#   denied         - Explicitly deny with reason
#   blocked        - Block entire command with reason
#
# All other keys are interpreted as subcommands.

# ============================================================
# UNRESTRICTED COMMANDS
# Allowed anywhere with any arguments.
# Just the command name with no value or empty value.
# ============================================================

cat:
head:
tail:
less:
ls:
pwd:
wc:
grep:
rg:
ag:
echo:
basename:
dirname:
sort:
uniq:
tr:
cut:
tee:
diff:
file:
stat:
which:
type:
env:
printenv:

# ============================================================
# BLOCKED COMMANDS
# Always denied. Use 'blocked' key with reason.
# ============================================================

rm:
  blocked: Deleting files is not permitted

rmdir:
  blocked: Deleting directories is not permitted

sudo:
  blocked: Elevated privileges not permitted

su:
  blocked: Switching users not permitted

curl:
  blocked: Network requests not permitted

wget:
  blocked: Network requests not permitted

nc:
  blocked: Network connections not permitted

netcat:
  blocked: Network connections not permitted

chmod:
  blocked: Changing permissions not permitted

chown:
  blocked: Changing ownership not permitted

mv:
  blocked: Moving/renaming files not permitted

dd:
  blocked: Low-level disk operations not permitted

mkfs:
  blocked: Filesystem operations not permitted

kill:
  blocked: Killing processes not permitted

pkill:
  blocked: Killing processes not permitted

killall:
  blocked: Killing processes not permitted

# ============================================================
# RESTRICTED COMMANDS
# Commands with subcommand/argument/directory restrictions.
# ============================================================

git:
  description: Git version control (read-only mode for code review)
  allowed-dirs:
    - ~/projects/trusted
    - ~/projects/trusted/**
    - ~/work/client-*
    - ~/work/client-*/**
  
  # --------------------------------------------------------
  # Allowed subcommands (any arguments)
  # --------------------------------------------------------
  log:
    description: View commit history
  
  diff:
    description: Show differences between commits, working tree, etc.
  
  show:
    description: Show various types of objects
  
  status:
    description: Show working tree status
  
  blame:
    description: Show what revision and author last modified each line
  
  shortlog:
    description: Summarize git log output
  
  rev-parse:
    description: Parse revision specifications
  
  ls-files:
    description: Show information about files in index and working tree
  
  ls-tree:
    description: List the contents of a tree object
  
  describe:
    description: Give an object a human readable name
  
  whatchanged:
    description: Show logs with difference each commit introduces
  
  # --------------------------------------------------------
  # Subcommands with argument restrictions
  # --------------------------------------------------------
  branch:
    description: List branches only (creating/deleting not allowed)
    allowed-args:
      - -l
      - -a
      - -v
      - -r
      - --list
      - --all
      - --remotes
      - -la
      - -lv
      - -av
      - -rav
      - --merged
      - --no-merged
      - --contains
  
  remote:
    description: View remotes only
    allowed-args:
      - -v
      - --verbose
    show:
      description: Show information about remote
  
  stash:
    description: View stashes only (modifying stashes not allowed)
    list:
      description: List stashed changes
    show:
      description: Show stash contents
  
  config:
    description: Read configuration only
    allowed-args:
      - --get
      - --get-all
      - --list
      - -l
      - --show-origin
      - --show-scope
  
  tag:
    description: List tags only
    allowed-args:
      - -l
      - --list
      - -n
  
  worktree:
    description: List worktrees only
    list:
      description: List worktrees
  
  # --------------------------------------------------------
  # Explicitly denied subcommands (with reasons)
  # --------------------------------------------------------
  commit:
    denied: Commits not allowed in review mode
  
  push:
    denied: Pushing not allowed - review only
  
  pull:
    denied: Pulling could change working state
  
  fetch:
    denied: Fetching could modify refs
  
  reset:
    denied: Reset could lose uncommitted work
  
  checkout:
    denied: Checkout could change working state
  
  switch:
    denied: Switch could change working state
  
  restore:
    denied: Restore could change working state
  
  merge:
    denied: Merging not allowed in review mode
  
  rebase:
    denied: Rebasing not allowed in review mode
  
  cherry-pick:
    denied: Cherry-pick not allowed in review mode
  
  revert:
    denied: Revert creates commits
  
  clean:
    denied: Clean deletes untracked files
  
  rm:
    denied: Removing files not allowed
  
  add:
    denied: Staging changes not allowed
  
  apply:
    denied: Applying patches not allowed
  
  am:
    denied: Applying mailbox patches not allowed
  
  clone:
    denied: Cloning repositories not allowed
  
  init:
    denied: Initializing repositories not allowed
  
  submodule:
    denied: Submodule operations not allowed

# --------------------------------------------------------
# FIND - Directory restricted
# --------------------------------------------------------
find:
  description: Find files (restricted to project directories)
  allowed-dirs:
    - ~/projects
    - ~/projects/**
    - ~/work
    - ~/work/**

# --------------------------------------------------------
# PYTHON - Limited operations
# --------------------------------------------------------
python:
  description: Python interpreter (limited safe operations)
  allowed-args:
    - --version
    - -V
    - -c
  -m:
    description: Run library module
    json.tool:
      description: JSON formatting/validation
    py_compile:
      description: Compile Python source files
    this:
      description: The Zen of Python
    calendar:
      description: Display calendar

python3:
  description: Python 3 interpreter (limited safe operations)
  allowed-args:
    - --version
    - -V
    - -c
  -m:
    description: Run library module
    json.tool:
    py_compile:
    this:
    calendar:

# --------------------------------------------------------
# NPM - Read-only operations
# --------------------------------------------------------
npm:
  description: Node package manager (read-only)
  allowed-args:
    - --version
    - -v
  
  list:
    description: List installed packages
  ls:
    description: List installed packages
  view:
    description: View package info
  info:
    description: View package info
  show:
    description: View package info
  outdated:
    description: Check for outdated packages
  audit:
    description: Run security audit
  explain:
    description: Explain installed packages
  why:
    description: Explain why a package is installed
  
  install:
    denied: Installing packages not allowed
  uninstall:
    denied: Uninstalling packages not allowed
  update:
    denied: Updating packages not allowed
  run:
    denied: Running scripts not allowed
  start:
    denied: Running scripts not allowed
  test:
    denied: Running scripts not allowed
  publish:
    denied: Publishing not allowed
  init:
    denied: Initializing projects not allowed

# --------------------------------------------------------
# YARN - Read-only operations
# --------------------------------------------------------
yarn:
  description: Yarn package manager (read-only)
  allowed-args:
    - --version
    - -v
  
  list:
  info:
  why:
  outdated:
  audit:
  
  add:
    denied: Adding packages not allowed
  remove:
    denied: Removing packages not allowed
  install:
    denied: Installing not allowed
  run:
    denied: Running scripts not allowed

# --------------------------------------------------------
# DOCKER - Inspection only
# --------------------------------------------------------
docker:
  description: Docker (inspection only)
  
  ps:
    description: List containers
  images:
    description: List images
  inspect:
    description: Inspect objects
  logs:
    description: View container logs
  version:
    description: Show version
  info:
    description: Display system info
  
  run:
    denied: Running containers not allowed
  exec:
    denied: Executing in containers not allowed
  rm:
    denied: Removing containers not allowed
  rmi:
    denied: Removing images not allowed
  build:
    denied: Building images not allowed
  push:
    denied: Pushing images not allowed
  pull:
    denied: Pulling images not allowed

# --------------------------------------------------------
# MAKE - Inspection only
# --------------------------------------------------------
make:
  description: Make (dry-run and inspection only)
  allowed-args:
    - -n
    - --dry-run
    - -p
    - --print-data-base
    - -q
    - --question
```

---

## 3. Command-Line Interface

### 3.1 Synopsis

```
run-command-safely.sh [OPTIONS] COMMAND [ARGS...]
run-command-safely.sh --allowed-commands [--verbose]
run-command-safely.sh --help [COMMAND [SUBCOMMAND...]]
run-command-safely.sh --version
```

### 3.2 Options

| Option | Short | Description |
|--------|-------|-------------|
| `--config FILE` | `-c` | Use alternate config file |
| `--allowed-commands` | | List all allowed commands (summary) |
| `--verbose` | `-v` | Show more detail (with --allowed-commands) |
| `--help [CMD...]` | `-h` | Show help, optionally for specific command path |
| `--dry-run` | `-n` | Check if command would be allowed without executing |
| `--why` | | Explain matching decision in detail |
| `--version` | | Show script version |

### 3.3 Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Command executed successfully (or --dry-run: would be allowed) |
| 1 | Command denied by policy |
| 2 | Command not found in system |
| 3 | Configuration error |
| 4 | Invalid usage |
| 126 | Command found but not executable |
| 127 | Command not found |
| 128+ | Command was killed by signal (128 + signal number) |

---

## 4. Matching Algorithm

### 4.1 High-Level Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    INPUT: command + args + cwd                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ 1. Lookup command   │
                   │    in config root   │
                   └──────────┬──────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
         NOT FOUND                         FOUND
              │                               │
              ▼                               ▼
        ┌───────────┐              ┌─────────────────────┐
        │  DENY     │              │ 2. Check 'blocked'  │
        │ (unknown) │              │    key              │
        └───────────┘              └──────────┬──────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              │                               │
                         HAS 'blocked'                   NO 'blocked'
                              │                               │
                              ▼                               ▼
                        ┌───────────┐              ┌─────────────────────┐
                        │  DENY     │              │ 3. Check directory  │
                        │ (blocked) │              │    restrictions     │
                        └───────────┘              └──────────┬──────────┘
                                                              │
                                              ┌───────────────┴───────────────┐
                                              │                               │
                                     HAS 'allowed-dirs'               NO 'allowed-dirs'
                                        AND NOT MATCH                    OR MATCH
                                              │                               │
                                              ▼                               ▼
                                        ┌───────────┐              ┌─────────────────────┐
                                        │  DENY     │              │ 4. Traverse args    │
                                        │ (dir)     │              │    through tree     │
                                        └───────────┘              └──────────┬──────────┘
                                                                              │
                                                                              ▼
                                                                   ┌─────────────────────┐
                                                                   │   TREE TRAVERSAL    │
                                                                   │   (see 4.2)         │
                                                                   └─────────────────────┘
```

### 4.2 Tree Traversal Algorithm

```
FUNCTION traverse_tree(node, remaining_args, path):
    # path tracks the matched command path for error messages
    
    IF remaining_args is empty:
        # No more args to match - command is allowed
        RETURN Allow(path)
    
    arg = remaining_args[0]
    rest = remaining_args[1:]
    
    # Check if arg matches a child node
    IF arg IN node.children:
        child = node.children[arg]
        new_path = path + [arg]
        
        # Check if explicitly denied
        IF child.has_key('denied'):
            RETURN Deny(
                reason = child.denied,
                path = new_path,
                allowed_siblings = get_allowed_children(node)
            )
        
        # Check directory restrictions (may override parent)
        IF child.has_key('allowed-dirs'):
            IF NOT matches_glob(cwd, child.allowed-dirs):
                RETURN Deny(
                    reason = "Not allowed in current directory",
                    path = new_path,
                    allowed_dirs = child.allowed-dirs
                )
        
        # Continue traversal
        RETURN traverse_tree(child, rest, new_path)
    
    # arg doesn't match any child - check allowed-args
    IF node.has_key('allowed-args'):
        IF arg IN node.allowed-args:
            # This arg is explicitly allowed
            # Continue checking remaining args against same node
            RETURN traverse_tree(node, rest, path)
        ELSE:
            # Arg not in allowed list
            RETURN Deny(
                reason = f"'{arg}' is not an allowed argument",
                path = path,
                allowed_args = node.allowed-args
            )
    
    # No children matched, no allowed-args restriction
    # This means any remaining args are allowed
    RETURN Allow(path)


FUNCTION check_command(args, cwd, config):
    IF args is empty:
        RETURN Error("No command provided")
    
    cmd = args[0]
    remaining = args[1:]
    
    IF cmd NOT IN config:
        RETURN Deny(
            reason = f"'{cmd}' is not a recognized command",
            hint = "Use --allowed-commands to see available commands"
        )
    
    node = config[cmd]
    
    # Check if blocked
    IF node.has_key('blocked'):
        RETURN Deny(
            reason = node.blocked,
            path = [cmd]
        )
    
    # Check directory restrictions at command level
    IF node.has_key('allowed-dirs'):
        IF NOT matches_glob(cwd, node.allowed-dirs):
            RETURN Deny(
                reason = f"'{cmd}' is not allowed in current directory",
                path = [cmd],
                allowe

... EOF
