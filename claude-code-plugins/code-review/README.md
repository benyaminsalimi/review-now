# code-review-skill

A Claude Code plugin that provides `/code_review` — a multi-agent code review command with branch comparison and interactive uncommitted changes review.

## Install

```bash
claude plugin add /path/to/agent_skill
```

## Usage

```
/code_review <source-branch> <target-branch>
/code_review uncommitted
```

### Branch Comparison

```
/code_review feature-branch main
```

Fetches both branches from origin, diffs them, and produces a structured review report with a verdict.

### Uncommitted Changes

```
/code_review uncommitted
```

Reviews all staged, unstaged, and untracked changes. After the review, walks through each finding one at a time with interactive options:

- **Fix** — applies the recommended fix
- **Show solution** — shows the code change before deciding
- **Skip** — move to next finding
- **Skip all** — stop walkthrough, list remaining findings

## How It Works

### Review Strategy Selection

The plugin picks its strategy automatically based on one file:

| `REVIEW_GUIDELINES.md` in repo root? | Strategy |
|---------------------------------------|----------|
| Yes | Reviews directly using your guidelines. No agents. |
| No | Dispatches 2 specialized agents in parallel. |

### Multi-Agent Pipeline (no guidelines file)

When no `REVIEW_GUIDELINES.md` is found, the plugin runs this pipeline:

```
Pre-flight checks (branches exist? changes exist?)
       |
Discover AGENTS.md files (scoped rules per directory)
       |
Gather diff + file list + commit log
       |
Launch 2 agents in parallel
  |- Security review agent (sonnet)
  |- General code review agent (sonnet)
       |
Deduplicate findings
       |
Validate each finding (parallel haiku agents)
  |- Reads actual code to confirm or reject
       |
Filter false positives
       |
Output report with verdict
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
agent_skill/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest
├── commands/
│   └── code_review.md        # Orchestrator (mode routing, agent dispatch, report)
├── agents/
│   ├── security-review.md    # Security agent prompt
│   └── general-review.md     # General review agent prompt
└── README.md
```

## Acknowledgements

This plugin was inspired by and built upon the work of:

- **Security Review Agent**: Based on [claude-code-security-review](https://github.com/anthropics/claude-code-security-review) by Anthropic
- **General Review Agent**: Inspired by [OpenAI Codex](https://github.com/openai/codex) review patterns
