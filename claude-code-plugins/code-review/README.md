# code-review

A Claude Code plugin providing interactive code review with multi-agent analysis. Two specialized commands for pull requests and uncommitted changes.

## Install

```bash
cd /path/to/my-ai-tools/claude-code-plugins/code-review
claude plugin marketplace add ./
claude plugin install code-review@local-dev
```

## Commands

### `/code_review:pr <source-branch> <target-branch>`

Interactive PR review with automatic commits for each fix.

```bash
/code_review:pr feature/user-auth main
```

**Features:**
- Fetches both branches and compares changes
- Multi-agent analysis (security + general + frontend)
- Interactive walkthrough with options for each finding
- Automatic git commits when fixes are applied

**Interactive Options:**
- **Apply fix** — applies recommended fix and commits automatically
- **Show solution** — preview the proposed change before deciding
- **Custom fix** — type or paste your own fix
- **Ask question** — ask deeper questions about the finding
- **Skip** — move to next finding
- **Skip all** — stop walkthrough, show summary

### `/code_review:uncommitted`

Interactive review of all uncommitted changes (no arguments needed).

```bash
/code_review:uncommitted
```

**Features:**
- Reviews staged + unstaged + untracked changes
- Multi-agent analysis (security + general + frontend)
- Interactive walkthrough with options for each finding
- Fixes applied to working directory (no auto-commit)

**Interactive Options:**
- **Apply fix** — applies recommended fix to working directory
- **Show solution** — preview the proposed change before deciding
- **Custom fix** — type or paste your own fix
- **Ask question** — ask deeper questions about the finding
- **Skip** — move to next finding
- **Skip all** — stop walkthrough, show summary

## How It Works

### Review Strategy Selection

Both commands automatically pick their strategy based on one file:

| `REVIEW_GUIDELINES.md` in repo root? | Strategy |
|---------------------------------------|----------|
| Yes | Reviews directly using your guidelines. No agents. |
| No | Dispatches 3 specialized agents in parallel. |

### Multi-Agent Pipeline (no guidelines file)

When no `REVIEW_GUIDELINES.md` is found, both commands run this pipeline:

```
Pre-flight checks (branches exist? changes exist?)
       |
Discover AGENTS.md files (scoped rules per directory)
       |
Gather diff + file list + commit log
       |
Launch 3 agents in parallel
  |- Security review agent (sonnet)
  |- General code review agent (sonnet)
  |- Frontend/Angular review agent (sonnet)
       |
Deduplicate findings
       |
Validate each finding (parallel haiku agents)
  |- Reads actual code to confirm or reject
       |
Filter false positives
       |
Interactive walkthrough
  |- Apply fix / Show solution / Custom fix / Ask question / Skip
  |- (PR mode: auto-commits after each fix)
       |
Final summary with verdict
```

### Guideline-Based Review (guidelines file found)

When `REVIEW_GUIDELINES.md` exists in the repo root, the orchestrator reviews the diff directly using your custom guidelines. No agents are launched. Put whatever review rules you want in that file and the plugin follows them exactly.

## Agents

### Security Review Agent

A senior security engineer persona focused on high-confidence vulnerabilities. Examines:

- Input validation (SQL injection, command injection, XXE, path traversal)
- Authentication & authorization (bypass, privilege escalation, JWT, session flaws)
- Crypto & secrets (hardcoded keys, weak algorithms, cert validation)
- Injection & code execution (deserialization, pickle, eval, XSS)
- Data exposure (PII, sensitive logging, debug info)

Uses a 3-phase methodology: repository context research, comparative analysis, vulnerability assessment.

Only reports findings with >80% confidence. Explicitly excludes DoS, rate limiting, secrets on disk, and speculative issues.

### General Review Agent

A code reviewer persona focused on correctness, maintainability, and quality. Flags issues that:

- Meaningfully impact accuracy, performance, security, or maintainability
- Were introduced in the reviewed changes (not pre-existing)
- Have provable impact on other parts of the code
- The author would fix if aware of them

Covers untrusted user input handling (open redirects, SQL parameterization, SSRF protection), dependency review, fail-fast patterns, error handling, and system-level thinking.

### Frontend/Angular Review Agent

An Angular expert persona focused on modern Angular best practices, performance, and architecture. Examines:

- Signal-based architecture (input(), model(), computed(), output())
- Change detection optimization (track expressions, class/style bindings)
- Modern template syntax (@if, @for, @switch instead of *ngIf, *ngFor)
- Dependency injection patterns (inject() function, DestroyRef)
- Lifecycle management (proper hook usage, render callbacks)
- Component API design (input/output naming, transforms)
- Content projection and composition
- View encapsulation and styling

Uses Angular official guidelines from angular.dev. Flags P0 issues like missing `track` expressions in `@for` loops, and P1 issues like using legacy syntax or improper signal patterns.

## Priority System

Both agents use the same P0-P3 scale:

| Priority | Meaning |
|----------|---------|
| **P0** | Drop everything. Blocking release/operations. |
| **P1** | Urgent. Fix in the next cycle. |
| **P2** | Normal. Fix eventually. |
| **P3** | Low. Nice to have. |

## Verdict

| Verdict | Condition |
|---------|-----------|
| **REQUEST CHANGES** | Any P0 or P1 findings |
| **NEEDS DISCUSSION** | Any P2 findings with confidence > 0.85 |
| **APPROVE** | Everything else |

## AGENTS.md Support

Place an `AGENTS.md` file in any directory to define scoped rules. The plugin discovers all `AGENTS.md` files in the repo and checks changed files against the rules in their scope (directory and subdirectories). Violations are flagged with the exact rule quoted.

## Validation Layer

Every finding from the review agents goes through a validation step:

1. A haiku-tier agent reads the actual file at the reported location
2. Confirms whether the issue genuinely exists in the code
3. Rejects false positives

After validation, a final filter discards findings that are pre-existing, style-only, linter-catchable, or depend on specific inputs/state.

## Output Schema

All agents produce the same JSON format for easy merging:

```json
{
  "findings": [
    {
      "file": "path/to/file.py",
      "line": 42,
      "priority": "P1",
      "category": "logic_error",
      "description": "...",
      "reason": "bug",
      "recommendation": "...",
      "confidence": 0.9
    }
  ],
  "analysis_summary": {
    "files_reviewed": 8,
    "p0_count": 0,
    "p1_count": 1,
    "p2_count": 0,
    "p3_count": 0,
    "verdict": "needs attention",
    "review_completed": true
  }
}
```

## File Structure

```
code-review/
├── .claude-plugin/
│   ├── plugin.json                      # Plugin manifest
│   └── marketplace.json                 # Marketplace configuration
├── commands/
│   ├── code_review_pr.md                # PR review orchestrator
│   └── code_review_uncommitted.md       # Uncommitted changes orchestrator
├── agents/
│   ├── security-review.md               # Security agent prompt
│   ├── general-review.md                # General review agent prompt
│   └── frontend-review.md               # Frontend/Angular agent prompt
└── README.md
```

## Acknowledgements

This plugin was inspired by and built upon the work of:

- **Security Review Agent**: Based on [claude-code-security-review](https://github.com/anthropics/claude-code-security-review) by Anthropic
- **General Review Agent**: Inspired by [OpenAI Codex](https://github.com/openai/codex) review patterns
