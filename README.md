# Codebase Quality Plugin

A high-signal code quality checker for Claude Code. Finds real issues, not noise.

## What It Checks

| Category | Agent | Model | Finds |
|----------|-------|-------|-------|
| **Secrets** | `secret-auditor` | Opus | Private keys, hardcoded tokens, tracked .env files |
| **Bugs** | `deterministic-bug-scan` | Opus | Merge conflict markers, malformed JSON, syntax errors |
| **Compliance** | `compliance-auditor` ×2 | Sonnet | CLAUDE.md rule violations (must quote exact rule) |
| **Logic** | `logic-quality-auditor` | Sonnet | Empty catch blocks, dead code, null safety, complexity |
| **Support** | `repo-profiler` | Haiku | Stack detection, smoke commands |
| **Support** | `claude-md-resolver` | Haiku | CLAUDE.md file discovery |
| **Support** | `tracked-env-detector` | Haiku | Suspicious tracked files (.env, *.pem) |

## What It Won't Flag

- Style/formatting preferences
- "Might be wrong" issues
- Missing tests
- Architectural opinions
- Anything requiring runtime context

## Installation

```bash
git clone https://github.com/baranbingol1/claude-codebase-quality.git
```

## Usage

Start Claude Code with the plugin:

```bash
claude --plugin-dir ./claude-codebase-quality
```

Then run the command:

```
/codebase-quality:codebase-quality
```

### Options

```
/codebase-quality:codebase-quality [paths...] [--fast|--smoke]
```

| Flag | Description |
|------|-------------|
| `--fast` | (default) Static checks, no installs |
| `--smoke` | Run dependency-free syntax/compile checks |
| `[paths]` | Scope to specific files/directories |

### Examples

```
/codebase-quality:codebase-quality                     # whole repo, fast mode
/codebase-quality:codebase-quality src/                # only src/ directory
/codebase-quality:codebase-quality --smoke             # include syntax checks
/codebase-quality:codebase-quality src/ lib/ --smoke   # scoped + smoke
```

## Output

Issues are grouped by severity:

- **Errors** — must fix (secrets, bugs, empty catch blocks)
- **Warnings** — should fix (complexity, suspicious patterns)
- **Info** — awareness (TODO/FIXME markers)

Each issue includes file path, line numbers, evidence snippet, and GitHub permalink (if applicable).

## License

MIT
