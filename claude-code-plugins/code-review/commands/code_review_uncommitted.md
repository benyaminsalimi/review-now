---
description: Interactive review of uncommitted changes with multi-agent analysis
allowed-tools: Bash(git:*), Read, Edit, Grep, Glob, Task, AskUserQuestion
---

# /code_review:uncommitted

Interactive code review for uncommitted changes. Reviews all staged, unstaged, and untracked changes with full walkthrough of each finding.

Agent assumptions:
- All tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

---

## Step 1: Pre-flight Check

Verify there are uncommitted changes to review:

```bash
git status --short
```

If clean working tree (no output), print "No uncommitted changes to review." and stop.

---

## Step 2: Load Review Guidelines

Check if `REVIEW_GUIDELINES.md` exists in the repository root using Glob.

- **If found** → Read it. Store as `$GUIDELINES`. Set `$REVIEW_MODE = "guidelines"`.
- **If not found** → Set `$REVIEW_MODE = "agents"`.

---

## Step 3: Discover AGENTS.md Files

Use Glob to find all `**/AGENTS.md` files. Read them and store as `$AGENTS_MD_RULES` with their directory paths.

---

## Step 4: Gather Changes

**Get unstaged changes:**
```bash
git diff
```

**Get staged changes:**
```bash
git diff --cached
```

**Get overview including untracked:**
```bash
git status --short
```

For untracked files, read their contents with Read tool.

Store combined diff as `$DIFF`, file list as `$FILES`.

---

## Step 5: Execute Review

### If `$REVIEW_MODE = "guidelines"` — Guideline-Based Review

Review the diff using `$GUIDELINES` and `$AGENTS_MD_RULES`. Output findings.

### If `$REVIEW_MODE = "agents"` — Multi-Agent Review

Launch ALL THREE agents in parallel:

1. **Security Review** (read `agents/security-review.md`, use Task with model "sonnet")
2. **General Review** (read `agents/general-review.md`, use Task with model "sonnet")
3. **Frontend Review** (read `agents/frontend-review.md`, use Task with model "sonnet")

Pass to each agent:
- DIFF: `$DIFF`
- FILES CHANGED: `$FILES`
- AGENTS.MD RULES: `$AGENTS_MD_RULES`

After all agents return:
1. Parse JSON from all agents
2. Deduplicate findings (same file+line+category = keep highest confidence)
3. Store as `$ALL_FINDINGS`

---

## Step 6: Validate Findings

For each finding, launch validation agent (Task with model "haiku"):

```
Prompt:
"You are a validation agent. Verify this reported issue:

REPORTED ISSUE:
- File: [file]
- Line: [line]
- Priority: [priority]
- Category: [category]
- Description: [description]

Read the file and check if the issue actually exists. Reply:
VALIDATED - [reason]
REJECTED - [reason]"
```

Launch all validation agents in parallel. Filter out REJECTED findings. Store as `$VALIDATED_FINDINGS`.

---

## Step 7: Interactive Walkthrough

**IMPORTANT**: Always run interactive walkthrough, even if findings are empty.

If no findings, print "✓ No issues found. Code looks good!" and stop.

For each finding in `$VALIDATED_FINDINGS` (ordered by priority P0 first):

### Present Finding

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Finding [N] of [total]: [priority] in [file]:[line]

Category: [category]
Confidence: [confidence]

Description:
[description]

Recommendation:
[recommendation]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Ask User Action

Use `AskUserQuestion` with these options:

```json
{
  "question": "How would you like to handle this finding?",
  "header": "Action",
  "options": [
    {
      "label": "Apply fix",
      "description": "Apply the recommended fix to your working directory"
    },
    {
      "label": "Show solution",
      "description": "Show the proposed code change before deciding"
    },
    {
      "label": "Custom fix",
      "description": "Type your own fix to apply"
    },
    {
      "label": "Ask question",
      "description": "Ask deeper questions about this finding"
    },
    {
      "label": "Skip",
      "description": "Skip this finding and move to the next"
    },
    {
      "label": "Skip all",
      "description": "Stop walkthrough, show summary of remaining findings"
    }
  ]
}
```

### Handle User Choice

**If "Apply fix":**
1. Read the target file with Read tool
2. Apply the fix using Edit tool
3. Confirm: "✓ Fix applied to working directory. Moving to next finding."
4. Move to next finding

**If "Show solution":**
1. Read the target file
2. Display the exact code change (old vs new)
3. Ask again with options: "Apply fix" / "Custom fix" / "Skip"

**If "Custom fix":**
1. Ask user: "Please describe or paste your fix:"
2. Use AskUserQuestion with free-text input
3. Apply the user's fix using Edit tool
4. Confirm: "✓ Custom fix applied. Moving to next finding."
5. Move to next finding

**If "Ask question":**
1. Ask user: "What would you like to know about this finding?"
2. Use AskUserQuestion with free-text input
3. Analyze the finding and code context to answer the question
4. After answering, ask again: "Apply fix" / "Custom fix" / "Skip"

**If "Skip":**
1. Print "Skipped. Moving to next finding."
2. Move to next finding

**If "Skip all":**
1. Print "Stopping walkthrough."
2. List all remaining findings with file:line and description
3. Print summary and stop

---

## Step 8: Final Summary

After walkthrough completes:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Uncommitted Changes Review Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Files changed:** [count]
**Review strategy:** [Multi-agent / Guidelines]

**Findings:**
- Total: [N]
- Fixed: [N]
- Skipped: [N]

**Priority breakdown:**
- P0: [N]
- P1: [N]
- P2: [N]
- P3: [N]

**Next steps:**
- Review changes: git diff
- Stage changes: git add .
- Commit: git commit -m "message"
```
