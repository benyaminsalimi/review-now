# review-now

Interactive code review plugin for Claude Code with multi-agent analysis and conversational walkthrough.

## What is review-now?

A Claude Code plugin that transforms code review from reading static reports into having interactive conversations. Walk through findings one at a time, apply fixes inline, ask questions, and move fast through your review queue.

Built for two scenarios:
1. **Reviewing team PRs** - Fast, interactive walkthrough of all changes
2. **Reviewing AI-generated code** - Catch issues in code you prompted AI to write

## Installation

### From GitHub (Recommended)

```bash
# Add the marketplace
claude plugin marketplace add benyaminsalimi/review-now

# Install the plugin
claude plugin install review-now@review-now-marketplace
```

### Local Development

```bash
git clone https://github.com/benyaminsalimi/review-now.git
cd review-now
claude plugin marketplace add ./
claude plugin install review-now@review-now-marketplace
```

## Commands

### `/review-now:pr <source-branch> <target-branch>`

Interactive PR review with automatic commits for each fix.

```bash
/review-now:pr feature/user-auth main
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

### `/review-now:uncommitted`

Interactive review of all uncommitted changes (no arguments needed).

```bash
/review-now:uncommitted
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

### Multi-Agent Review Pipeline

```
Pre-flight checks
       |
Discover custom guidelines (REVIEW_GUIDELINES.md, AGENTS.md)
       |
Gather changes (diff, files, commits)
       |
Launch 3 agents in parallel
  |- Security review agent (Anthropic guidelines)
  |- General review agent (Codex patterns)
  |- Frontend/Angular review agent (Angular official docs)
       |
Deduplicate findings
       |
Validate with lightweight agents (filter false positives)
       |
Interactive walkthrough
  |- Apply fix / Show solution / Custom fix / Ask question / Skip
  |- (PR mode: auto-commits after each fix)
       |
Final summary with verdict
```

### Review Strategy

The plugin automatically picks its strategy:

| `REVIEW_GUIDELINES.md` in repo root? | Strategy |
|---------------------------------------|----------|
| Yes | Reviews using your custom guidelines. No agents. |
| No | Dispatches 3 specialized agents in parallel. |

## Agents

### Security Review Agent
Based on [Anthropic's security review guidelines](https://github.com/anthropics/claude-code-security-review). Focuses on high-confidence vulnerabilities: SQL injection, authentication bypasses, hardcoded secrets, crypto issues, XSS, and data exposure.

### General Review Agent
Adapted from [OpenAI Codex review patterns](https://github.com/openai/codex). Focuses on correctness, maintainability, logic errors, error handling, and code quality.

### Frontend/Angular Review Agent
Built from [Angular's official LLM context](https://angular.dev/assets/context/llms-full.txt). Focuses on signal-based architecture, change detection, modern template syntax, component patterns, and performance.

## Priority System

All agents use P0-P3 scale:

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

## Custom Guidelines

### REVIEW_GUIDELINES.md

Place a `REVIEW_GUIDELINES.md` file in your repo root to define custom review rules. The plugin will use your guidelines instead of running agents.

### AGENTS.md

Place `AGENTS.md` files in any directory to define scoped rules. The plugin discovers all `AGENTS.md` files and checks changed files against rules in their scope (directory and subdirectories).

## Building Custom Agents

See the `agents/` directory for examples. Each agent is a markdown file with:
- Frontmatter (name, description, model, tools)
- System prompt with review guidelines
- JSON output schema

Create domain-specific agents for your tech stack:
- Backend frameworks (Django, Rails, Spring)
- Mobile (iOS, Android, React Native)
- Infrastructure (Terraform, Kubernetes)
- Database patterns
- Security models specific to your org

## File Structure

```
review-now/
├── .claude-plugin/
│   ├── plugin.json                      # Plugin manifest
│   └── marketplace.json                 # Marketplace configuration
├── commands/
│   ├── review_now_pr.md                 # PR review orchestrator
│   └── review_now_uncommitted.md        # Uncommitted changes orchestrator
├── agents/
│   ├── security-review.md               # Security agent
│   ├── general-review.md                # General review agent
│   └── frontend-review.md               # Frontend/Angular agent
├── LICENSE
└── README.md
```

## Contributing

Contributions welcome! We're open to:

**Code Contributions:**
- Additional language/framework agents (Python, Go, Rust, etc.)
- Better false positive filtering
- Custom validation rules
- Web UI for review management
- Performance improvements

**Hooks:**
- Pre-commit hooks for code review
- Post-review automation hooks
- Custom workflow integrations
- Share your hooks in discussions!

**Issues & Discussions:**
- Bug reports
- Feature requests
- Agent improvement suggestions
- Share your use cases and custom agents

Open an issue or PR on [GitHub](https://github.com/benyaminsalimi/review-now)!

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Acknowledgements

This plugin was inspired by and built upon:
- **Security Agent**: [claude-code-security-review](https://github.com/anthropics/claude-code-security-review) by Anthropic
- **General Review Agent**: [OpenAI Codex](https://github.com/openai/codex) review patterns
- **Frontend Agent**: [Angular LLM Context](https://angular.dev/assets/context/llms-full.txt)
