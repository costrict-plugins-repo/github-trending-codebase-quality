---
allowed-tools: Bash(git:*), Bash(rg:*), Bash(find:*), Bash(ls:*), Bash(cat:*), Bash(head:*), Bash(tail:*), Bash(wc:*), Bash(xargs:*), Bash(sed:*), Bash(awk:*), Bash(sort:*), Bash(uniq:*), Bash(cut:*), Bash(tr:*), Bash(python:*), Bash(python3:*), Bash(node:*), Bash(ruby:*), Bash(php:*), Bash(bash:*)
description: High-signal quality check of the current codebase (not a PR)
---

Provide a **high-signal quality check** for the **current codebase** (working tree / HEAD). This is NOT a PR review.

Use `$ARGUMENTS` to optionally scope the audit:

- Paths: any non-flag args are treated as repo-relative paths (files/dirs). If none are provided, audit the whole repo.
- Flags:
  - `--fast` (default): deterministic static checks + targeted scans. **No installs.**
  - `--smoke`: run only **dependency-free** syntax/compile checks (and only if they can run without installing deps). If tooling/deps are missing, report "could not run" (NOT a code issue).

If both `--fast` and `--smoke` are present, treat mode as `--smoke`.

**Global constraints (applies to orchestrator + all agents + all subagents):**
- All tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.
- Do NOT modify code files. Only write the final report file.
- Do NOT install dependencies.
- Do NOT run commands that dump environment variables (no `env`, `printenv`, `set`, etc.).
- Do NOT read `.env` / `.env.*` contents (paths-only is OK).

Create a todo list before starting.

---

## Procedure

### 0) Todo list
Create a todo list that mirrors the steps below. Keep it updated as you progress.

### 1) Repo context + scope (repo-profiler)
Launch the **`repo-profiler`** subagent (model: haiku) to gather minimal repo context and resolve scope.

It must return:
- repo root (from `git rev-parse --show-toplevel`)
- HEAD SHA (from `git rev-parse HEAD`) — full sha
- origin remote URL if present (from `git remote get-url origin`)
- resolved mode: `fast` or `smoke`
- resolved scope paths (repo-relative; empty = whole repo)
- detected stacks + likely entrypoints
- if mode == smoke: suggested **dependency-free** smoke commands (do not install deps)

**Stop condition:**
- If not a git repository, stop and output: "Not a git repository; cannot compute stable code links."

### 2) Relevant CLAUDE.md paths (claude-md-resolver)
Launch the **`claude-md-resolver`** subagent (model: haiku) to return a list of relevant CLAUDE.md file paths (paths only; no contents):
- include root `CLAUDE.md` if present
- include any `CLAUDE.md` under the audit scope (if scoped), otherwise anywhere in the repo

**Scoping rule used later:**
A `CLAUDE.md` applies only to files in its directory and descendant directories.

### 3) Codebase summary (sonnet summary agent)
Launch a **sonnet** agent to produce a short summary of "what is here" using:
- the repo-profiler output (stacks/entrypoints)
- lightweight file inspection as needed (but no installs)
Return:
- brief description of the codebase structure
- what will be checked in this run (based on mode + scope)
- what will NOT be checked (to avoid false positives)

### 4) Parallel audit (high-signal only)
Launch these audits in parallel. Each must return a list of issues with:
- title
- description
- severity (for logic issues): error | warning | info
- reason flagged (one of: "CLAUDE.md adherence", "definite bug", "compile/syntax failure", "definite secret", "logic quality")
- file path(s) + exact line range(s) (or command + failure signature for smoke checks)
- minimal evidence (only the lines needed)
- if applicable: the exact CLAUDE.md rule quoted verbatim (and its path)

#### 4A) Compliance auditor #1 (compliance-auditor)
Launch **`compliance-auditor`** (model: sonnet).
- Only flag **clear, unambiguous** CLAUDE.md violations.
- Must quote exact rule text and point to exact violating lines.
- Only consider CLAUDE.md files applicable by directory ancestry.

#### 4B) Compliance auditor #2 (compliance-auditor)
Launch a second independent **`compliance-auditor`** (model: sonnet) with the same constraints.

#### 4C) Deterministic bug scan (deterministic-bug-scan)
Launch **`deterministic-bug-scan`** (model: opus).
- Focus on deterministic failures:
  - merge conflict markers: `<<<<<<<`, `=======`, `>>>>>>>`
  - malformed JSON only when validated deterministically (e.g., python json tool)
  - if mode == smoke: run ONLY the suggested dependency-free smoke commands
- Do NOT flag anything that depends on runtime inputs/state.

#### 4D) Tracked env/key path detection (tracked-env-detector) + secret audit (secret-auditor)
1) Launch **`tracked-env-detector`** (model: haiku) to produce **paths-only** lists of suspicious *tracked* files (no content reads).
2) Then launch **`secret-auditor`** (model: opus) with:
   - scope paths
   - head sha + origin url (for links if GitHub)
   - tracked suspicious paths from tracked-env-detector

**Critical constraints for secret-auditor:**
- Must NOT read environment variables.
- Must NOT open `.env` contents.
- Must NOT output secret values; redact evidence (`<REDACTED>`).
- Must respect env-var secret handling patterns (do NOT flag `process.env.*`, `os.environ[...]`, etc.).

#### 4E) Logic quality audit (logic-quality-auditor)
Launch **`logic-quality-auditor`** (model: sonnet).
- Flag high-confidence logic issues only:
  - **Error handling**: empty catch/except blocks, swallowed errors
  - **Dead code**: unreachable code after return/throw, large commented-out blocks (>10 lines)
  - **Null safety**: obvious null dereference without guards, inconsistent optional chaining
  - **Complexity**: functions >150 lines, nesting depth 5+, functions with 8+ parameters
  - **Suspicious patterns**: console.log in production code, magic numbers, infinite loop risks
  - **TODO/FIXME/HACK/XXX markers**: report as informational (not blocking)
- Do NOT flag style preferences, naming conventions, or "might be wrong" issues.
- Classify severity: error (must fix) | warning (should fix) | info (awareness)

---

### 5) Validate issues (parallel validators)
For each issue found in step 4, launch parallel validation subagents to confirm the issue is real with high confidence.

**Validator requirements (all types):**
- Confirm exact file + line range
- Confirm the evidence supports the claim
- If anything is uncertain, mark "NOT VALIDATED"

**Which validator to use:**
- For "CLAUDE.md adherence" issues: launch a **sonnet** validator (you may reuse `compliance-auditor` behavior) and confirm:
  - the quoted rule exists
  - the CLAUDE.md applies by ancestry
  - the violating line(s) actually violate the rule
- For "definite bug" / "compile/syntax failure" issues: launch an **opus** validator (you may reuse deterministic-bug-scan behavior) and confirm deterministically
- For "definite secret" issues: launch an **opus** validator (you may reuse secret-auditor behavior) and confirm:
  - it is not documentation/example text
  - evidence remains redacted
  - `.env` contents were not opened and env vars were not read
- For "logic quality" issues: launch a **sonnet** validator and confirm:
  - the issue is objectively problematic (not style preference)
  - the evidence matches the claim
  - the severity classification is appropriate

### 6) Filter to validated issues only
Discard any issue not validated in step 5. The remaining list is the final "high-signal issues" set.

### 7) Write report to file
After generating the report content, save it to a markdown file at the repo root:

- **Filename format**: `{folder_name}_{short_sha}_review_report.md`
  - `folder_name`: basename of repo root (e.g., `xatu-dev`)
  - `short_sha`: first 8 characters of HEAD SHA (from step 1)
- **Location**: repo root directory
- **Example**: `xatu-dev_a1b2c3d4_review_report.md`

Use the Write tool to save the complete report (everything from "## Codebase quality check" through the end of the report).

---

## Output requirements

### If issues exist: output a Codebase quality report

Use this exact format:

---

## Codebase quality check

### Scope
- Repo root: <path>
- HEAD: <full sha>
- Origin: <remote url or "none">
- Paths: <paths or "(entire repo)">
- Mode: fast|smoke

### Summary
<1 short paragraph>

### High-signal issues

#### Errors (must fix)
For each error-severity issue:
- **Title**
- **Category**: secret | bug | compliance | logic
- **Why it's a problem** (1–2 sentences)
- **Evidence**
  - `<file>:Lx-Ly`
  - minimal snippet (redacted if secret)
- **Link** (ONLY if origin is a GitHub remote and you can form OWNER/REPO)
  - Must use full sha:
    `https://github.com/OWNER/REPO/blob/<FULL_SHA>/<path>#L<start>-L<end>`
  - Provide at least 1 line of context before and after (centered on the issue)

#### Warnings (should fix)
For each warning-severity issue (same format as errors)

#### Info (awareness)
For each info-severity issue (same format, but grouped—e.g., "12 TODO markers found"):
- List locations briefly or summarize count by category

### Notes
- What was intentionally not checked (to avoid false positives)

---

### If NO issues: output this exact block

---

## Codebase quality check

No issues found. Checked for deterministic bugs, high-confidence secrets, CLAUDE.md compliance, and logic quality issues.

---

## False-positive avoidance rules (strict)
Do NOT flag:
- Style/formatting preferences
- "Might be wrong" or input/state-dependent issues
- Linter-only concerns (do not run linters to verify)
- General security advice without a concrete committed secret
- Anything that requires reading env vars or `.env` contents
- Naming convention violations (unless in CLAUDE.md)
- Missing tests or low test coverage
- Architectural opinions or "could be refactored" suggestions