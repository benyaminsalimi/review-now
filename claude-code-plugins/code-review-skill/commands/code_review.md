---
description: Code review -- branch diff or uncommitted changes
argument-hint: "<source> <target>" | "uncommitted"
allowed-tools: Bash(git:*), Bash(test:*), Bash(cat:*), Read, Edit, Grep, Glob, Task, AskUserQuestion
---

# /code_review

You are an orchestrator for code review. Depending on whether the repo has a `REVIEW_GUIDELINES.md`, you either review directly using those guidelines or dispatch specialized agents in parallel.

Agent assumptions (applies to all agents and subagents):
- All tools are functional and will work without error. Do not test tools or make exploratory calls. Make sure this is clear to every subagent that is launched.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

---

## Step 1: Parse Arguments

The user invoked `/code_review $ARGUMENTS`.

Determine the mode:

1. **No arguments** (empty `$ARGUMENTS`) → print usage help and stop
2. **`uncommitted`** → uncommitted changes mode
3. **Exactly two space-separated tokens** (e.g. `feature-branch main`) → branch comparison mode where `$1` is the source branch and `$2` is the target branch
4. **Anything else** → print usage help and stop (invalid input)

---

## Step 2: Usage Help

If no arguments or invalid arguments were provided, print this and stop:

```
### /code_review

**Usage:**

  /code_review <source-branch> <target-branch>
    Compare two branches. Fetches latest, diffs, and reviews.
    Example: /code_review feature-branch main

  /code_review uncommitted
    Review all uncommitted changes (staged + unstaged + untracked).
    Walks through each issue interactively: Fix / Show solution / Skip / Skip all.

**Review strategy:**
  - If REVIEW_GUIDELINES.md exists in repo root → reviews using those guidelines directly.
  - If no guidelines file → dispatches security + general review agents in parallel.
```

---

## Step 3: Pre-flight Checks

Before proceeding, verify there is something to review:

**For branch comparison mode:**
```bash
git fetch origin $1 $2
```
Then check that the branches exist and there is a diff between them. If no diff, print "No differences found between $1 and $2." and stop.

**For uncommitted mode:**
```bash
git status --short
```
If clean working tree (no output), print "No uncommitted changes to review." and stop.

---

## Step 4: Load Review Guidelines

Check if `REVIEW_GUIDELINES.md` exists in the repository root:

```
Use Glob to check for REVIEW_GUIDELINES.md in the repo root.
```

- **If found** → read it with `Read` tool. Store contents as `$GUIDELINES`. Set `$REVIEW_MODE = "guidelines"`.
- **If not found** → set `$REVIEW_MODE = "agents"`.

---

## Step 5: Discover AGENTS.md Files

Search for any AGENTS.md files that are relevant to the changed files:

1. Check for a root `AGENTS.md` file.
2. For each directory that contains a modified file, check if a `AGENTS.md` exists in that directory or any parent directory.

```
Use Glob to find all AGENTS.md files: **/AGENTS.md
```

Read all discovered AGENTS.md files. Store their contents as `$AGENTS_MD_RULES` along with the directory paths they apply to (an AGENTS.md only applies to files in its directory and subdirectories).

---

## Step 6: Gather Changes

### Mode A: Branch Comparison (`/code_review feature main`)

1. **Get the diff:**
   ```bash
   git diff origin/$2...origin/$1
   ```

2. **Get commit log:**
   ```bash
   git log --oneline origin/$2..origin/$1
   ```

3. **Get changed file list:**
   ```bash
   git diff --name-only origin/$2...origin/$1
   ```

Store the diff as `$DIFF`, file list as `$FILES`, commit log as `$LOG`.

### Mode B: Uncommitted Changes (`/code_review uncommitted`)

1. **Gather all changes:**
   ```bash
   git diff                    # unstaged changes
   git diff --cached           # staged changes
   git status --short          # overview including untracked
   ```

2. **For untracked files**, read their contents with `Read` tool.

Store the combined diff as `$DIFF`, file list as `$FILES`.

---

## Step 7: Execute Review

### If `$REVIEW_MODE = "guidelines"` — Guideline-Based Review

You perform the review yourself, directly. Do NOT dispatch agents.

Using the `$GUIDELINES` from `REVIEW_GUIDELINES.md`, review the `$DIFF` and `$FILES` by following every instruction in the guidelines document exactly. Apply the guidelines as your review criteria.

Also check changes against any `$AGENTS_MD_RULES` — for each changed file, check if an AGENTS.md applies and flag any violations. When flagging an AGENTS.md violation, quote the exact rule being broken and link to the AGENTS.md file path.

Read the changed files fully (not just the diff) using `Read` tool so you understand context around the changes.

Output your findings using the **Report Format** in Step 9, with a single unified "Findings" section. Use whatever priority/severity system the guidelines define. If the guidelines define no priority system, use P0-P3.

Then proceed to **Step 8** (interactive walkthrough) if in uncommitted mode, or output the report if in branch comparison mode.

### If `$REVIEW_MODE = "agents"` — Multi-Agent Review

Launch BOTH agents in parallel using the `Task` tool.

**IMPORTANT**: Launch both Task calls in the same message so they run in parallel.

#### Agent 1: Security Review (use model "sonnet")

Read the file `agents/security-review.md` from the plugin directory first. Then launch:

```
Task with subagent_type "general-purpose", model "sonnet".

Prompt:
"All tools are functional and will work without error. Do not test tools or make exploratory calls. Only call a tool if it is required.

You are the security review agent. Follow these instructions exactly:

[contents of agents/security-review.md]

CHANGES TO REVIEW:

DIFF:
[$DIFF]

FILES CHANGED:
[$FILES]

COMMIT LOG:
[$LOG if available]

AGENTS.md RULES (check for violations in changed files):
[$AGENTS_MD_RULES if any, else 'None found']

Output your findings as JSON matching the schema in your instructions."
```

#### Agent 2: General Code Review (use model "sonnet")

Read the file `agents/general-review.md` from the plugin directory first. Then launch:

```
Task with subagent_type "general-purpose", model "sonnet".

Prompt:
"All tools are functional and will work without error. Do not test tools or make exploratory calls. Only call a tool if it is required.

You are the general code review agent. Follow these instructions exactly:

[contents of agents/general-review.md]

CHANGES TO REVIEW:

DIFF:
[$DIFF]

FILES CHANGED:
[$FILES]

COMMIT LOG:
[$LOG if available]

AGENTS.md RULES (check for violations in changed files):
[$AGENTS_MD_RULES if any, else 'None found']

Output your findings as JSON matching the schema in your instructions."
```

### After All Agents Return

1. Parse the JSON from all agents.
2. Deduplicate findings — if multiple agents flag the same file+line+category, keep the one with highest confidence.
3. Collect all unique findings into `$ALL_FINDINGS`.

---

## Step 7b: Validate Findings

For each finding in `$ALL_FINDINGS`, launch a parallel validation subagent to confirm the issue is real.

```
Task with subagent_type "general-purpose", model "haiku".

Prompt:
"All tools are functional and will work without error. Only call a tool if required.

You are a validation agent. Your job is to verify whether the following reported issue is truly a real problem with high confidence.

REPORTED ISSUE:
- File: [file]
- Line: [line]
- Priority: [priority]
- Category: [category]
- Description: [description]
- Reason: [reason]

Read the file at the reported location using the Read tool. Check whether the stated issue actually exists in the code. Consider:
- Is the issue real or a false positive?
- Does the code actually do what the issue claims?
- For AGENTS.md violations: is the rule actually scoped for this file and actually violated?

Reply with exactly one of:
VALIDATED - [brief explanation why this is a real issue]
REJECTED - [brief explanation why this is a false positive]"
```

**Launch all validation agents in parallel.** After they return, filter out any findings that were REJECTED.

### False Positives Filter

Even after validation, discard any findings that match these patterns (these are known false positives):

- Pre-existing issues that were not introduced in this diff
- Something that appears to be a bug but is actually correct behavior
- Pedantic nitpicks that a senior engineer would not flag
- Issues that a linter would catch (do not run the linter to verify)
- General code quality concerns (e.g., lack of test coverage) unless explicitly required in AGENTS.md
- Issues mentioned in AGENTS.md but explicitly silenced in the code (e.g., via a lint ignore comment)
- Code style or quality concerns
- Potential issues that depend on specific inputs or state
- Subjective suggestions or improvements

Store surviving findings as `$VALIDATED_FINDINGS`.

---

## Step 8: Interactive Walkthrough (Uncommitted Mode Only)

This step applies ONLY when the mode is `uncommitted`, regardless of review mode.

If there are no findings, skip to Step 9.

After generating the review findings, walk through each finding ONE AT A TIME, ordered by priority (P0 first). For each finding, present:

```
### Finding [N/total]: [priority] in file:line
**Category**: ...
**Reason**: ...
**Description**: ...
**Recommendation**: ...

What would you like to do?
```

Then use `AskUserQuestion` with these options:
- **Fix** — Apply the recommended fix using `Edit` tool
- **Show solution** — Show the concrete code change, then ask again (Fix / Skip)
- **Skip** — Move to next finding
- **Skip all** — Stop walkthrough, show remaining findings as list

When the user chooses **Fix**:
- Read the target file with `Read` tool
- Apply the fix using `Edit` tool
- Confirm the fix was applied
- Move to the next finding

---

## Step 9: Output Report

### For Agent-Based Reviews (`$REVIEW_MODE = "agents"`)

```markdown
## Code Review Summary

| Field | Value |
|-------|-------|
| **Mode** | [branch comparison / uncommitted] |
| **Scope** | [branches compared / files changed count] |
| **Review strategy** | Multi-agent (security + general) |
| **Agents** | 2 review agents + validation |

---

### Findings

[For each validated finding, ordered by priority:]

#### [priority] — [category] in `file:line`
- **Reason**: [security / bug / AGENTS.md adherence]
- **Description**: ...
- **Recommendation**: ...
- **Confidence**: ...

[If no findings: "No issues found. Checked for bugs, security issues, and AGENTS.md compliance."]

---

### Overall Verdict

**[APPROVE / REQUEST CHANGES / NEEDS DISCUSSION]**

- P0: [N], P1: [N], P2: [N], P3: [N]

[Verdict logic:]
- **REQUEST CHANGES** if any P0 or P1 findings
- **NEEDS DISCUSSION** if any P2 findings with confidence > 0.85
- **APPROVE** otherwise
```

### For Guideline-Based Reviews (`$REVIEW_MODE = "guidelines"`)

```markdown
## Code Review Summary

| Field | Value |
|-------|-------|
| **Mode** | [branch comparison / uncommitted] |
| **Scope** | [branches compared / files changed count] |
| **Review strategy** | Guideline-based (REVIEW_GUIDELINES.md) |

---

### Findings

[For each finding, ordered by priority:]

#### [priority] — [category] in `file:line`
- **Description**: ...
- **Recommendation**: ...

[If no findings: "No issues found. Code looks good."]

---

### Overall Verdict

**[APPROVE / REQUEST CHANGES / NEEDS DISCUSSION]**

- Findings: [N] by priority breakdown

[Verdict logic:]
- **REQUEST CHANGES** if any P0/P1 findings
- **NEEDS DISCUSSION** if any P2 findings with high confidence
- **APPROVE** otherwise
```
