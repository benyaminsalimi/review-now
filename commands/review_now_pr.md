---
description: Interactive PR code review with multi-agent analysis
argument-hint: <source-branch> <target-branch>
allowed-tools: Bash(git:*), Read, Edit, Grep, Glob, Task, AskUserQuestion
---

# /review-now:pr

Interactive code review for pull requests. Reviews changes between two branches with full walkthrough of each finding.

Agent assumptions:
- All tools are functional and will work without error. Do not test tools or make exploratory calls.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

---

## Step 1: Parse Arguments

The user invoked `/review-now:pr $ARGUMENTS`.

**Required**: Exactly two space-separated arguments: `<source-branch> <target-branch>`

Example: `/review-now:pr feature/user-auth main`

If arguments are missing or invalid, print usage help and stop:

```
### /review-now:pr

Compare and review changes between two branches interactively.

**Usage:**
  /review-now:pr <source-branch> <target-branch>

**Example:**
  /review-now:pr feature/user-auth main

**Interactive walkthrough:**
  - Apply proposed fix (commits automatically)
  - Show solution before deciding
  - Enter custom fix
  - Ask deeper questions
  - Skip to next finding
```

---

## Step 2: Pre-flight Checks

Fetch both branches and verify there are changes to review:

```bash
git fetch origin $SOURCE_BRANCH $TARGET_BRANCH
```

Check that both branches exist and have differences:
```bash
git diff origin/$TARGET_BRANCH...origin/$SOURCE_BRANCH
```

If no differences found, print "No changes between $SOURCE_BRANCH and $TARGET_BRANCH." and stop.

---

## Step 3: Load Review Guidelines

Check if `REVIEW_GUIDELINES.md` exists in the repository root using Glob.

- **If found** → Read it. Store as `$GUIDELINES`. Set `$REVIEW_MODE = "guidelines"`.
- **If not found** → Set `$REVIEW_MODE = "agents"`.

---

## Step 4: Discover AGENTS.md Files

Use Glob to find all `**/AGENTS.md` files. Read them and store as `$AGENTS_MD_RULES` with their directory paths.

---

## Step 5: Gather Changes

**Get the diff:**
```bash
git diff origin/$TARGET_BRANCH...origin/$SOURCE_BRANCH
```

**Get commit log:**
```bash
git log --oneline origin/$TARGET_BRANCH..origin/$SOURCE_BRANCH
```

**Get changed file list:**
```bash
git diff --name-only origin/$TARGET_BRANCH...origin/$SOURCE_BRANCH
```

Store as `$DIFF`, `$LOG`, `$FILES`.

---

## Step 6: Execute Review

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
- COMMIT LOG: `$LOG`
- AGENTS.MD RULES: `$AGENTS_MD_RULES`

After all agents return:
1. Parse JSON from all agents
2. Deduplicate findings (same file+line+category = keep highest confidence)
3. Store as `$ALL_FINDINGS`

---

## Step 7: Validate Findings

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

## Step 8: Interactive Walkthrough

**IMPORTANT**: PR mode ALWAYS has interactive walkthrough, even if findings are empty.

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
      "description": "Apply the recommended fix and commit automatically"
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
3. Stage the file: `git add [file]`
4. Commit with descriptive message:
   ```bash
   git commit -m "Fix [priority]: [brief description]

   [Full description and recommendation]

   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```
5. Confirm: "✓ Fix applied and committed. Moving to next finding."
6. Move to next finding

**If "Show solution":**
1. Read the target file
2. Display the exact code change (old vs new)
3. Ask again with options: "Apply fix" / "Custom fix" / "Skip"

**If "Custom fix":**
1. Ask user: "Please describe or paste your fix:"
2. Use AskUserQuestion with free-text input
3. Apply the user's fix using Edit tool
4. Stage and commit:
   ```bash
   git add [file]
   git commit -m "Fix [priority]: [user's description]

   Custom fix for: [description]

   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```
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

## Step 9: Final Summary

After walkthrough completes:

```markdown
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
PR Review Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Branches:** [source] → [target]
**Files changed:** [count]
**Review strategy:** [Multi-agent / Guidelines]

**Findings:**
- Total: [N]
- Fixed & committed: [N]
- Skipped: [N]

**Priority breakdown:**
- P0: [N]
- P1: [N]
- P2: [N]
- P3: [N]

**Verdict:** [APPROVE / REQUEST CHANGES / NEEDS DISCUSSION]

**Next steps:**
- Review commits: git log
- Push changes: git push
- Create/update PR with findings
```
