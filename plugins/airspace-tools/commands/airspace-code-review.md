---
allowed-tools: Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git rev-parse:*), Bash(git branch:*), Bash(git fetch:*), Bash(git checkout:*), Bash(git diff:*), Bash(git log:*), Bash(cat review-*), mcp__github_inline_comment__create_inline_comment, mcp__atlassian__getAccessibleAtlassianResources, mcp__atlassian__getJiraIssue, AskUserQuestion
description: JIRA-aware code review for Airspace pull requests
argument-hint: "[free text context] [--pr=NUMBER] [--terminal|--comment|--markdown]"
---

Provide a JIRA-aware code review for a pull request or local branch changes.

**Arguments:** $ARGUMENTS

**Parse the arguments as follows:**

- `--pr=NUMBER` → review that specific PR. If omitted, run `gh pr view --json number` to find a PR for the current branch. If no PR exists, review local changes via `git diff main...HEAD`.
- `--terminal` or no output flag → print findings inline in the terminal (this is the default)
- `--comment` → post inline comments on the GitHub PR (requires a PR)
- `--markdown` → write findings to a `.md` file in the repo root
- Any remaining text (not a flag) is **user-provided context** — additional description, requirements, or comments to consider during the review. Pass this context to all review agents alongside the JIRA context.

**Agent assumptions (applies to all agents and subagents):**
- All tools are functional and will work without error. Do not test tools or make exploratory calls. Make sure this is clear to every subagent that is launched.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

**CRITICAL CONSTRAINTS (NEVER violate):**
- To fetch JIRA ticket data, you MUST use `mcp__atlassian__getAccessibleAtlassianResources` then `mcp__atlassian__getJiraIssue`. Do NOT use WebFetch, Fetch, or any HTTP tool to access JIRA. The Atlassian MCP tools handle authentication automatically.

To do this, follow these steps precisely:

1. **Determine PR context and announce the review.**
   - If `--pr=NUMBER` was provided, use that PR number.
   - Otherwise, run `gh pr view --json number` to check if the current branch has an open PR.
   - If a PR is found, store the PR number. If no PR exists, this is a **local branch review** — use `git diff main...HEAD` for the diff instead of PR diff. Note: `--comment` mode is not available without a PR.

   Print a single line to the terminal confirming the mode:
   - Terminal mode with PR: `Reviewing PR #<number> — printing results to terminal`
   - Terminal mode without PR: `Reviewing <branch name> (no PR) — printing results to terminal`
   - Comment mode: `Reviewing PR #<number> — posting comments to GitHub`
   - Markdown mode with PR: `Reviewing PR #<number> — writing results to review-PR-<number>.md`
   - Markdown mode without PR: `Reviewing <branch name> — writing results to review-<branch name>.md`

2. **Check for previous reviews (re-review detection).**

   Look for prior review feedback from these sources:
   - **Markdown files**: Check if a `review-PR-<number>.md` or `review-<branch>.md` file exists in the repo root.
   - **GitHub PR comments** (if PR exists): Check `gh pr view <PR> --comments` for previous review comments left by Claude.

   If previous feedback is found, this is a **re-review**. Collect all prior issues/feedback and store them. Print: `Re-review detected — will check if previous issues have been addressed`.

   If no previous feedback is found, this is a first review. Proceed normally.

3. **Skip this step if there is no PR.** Launch a haiku agent to check if any of the following are true:
   - The pull request is closed
   - The pull request is a draft
   - The pull request does not need code review (e.g. automated PR, trivial change that is obviously correct)

   If any condition is true, stop and do not proceed.

Note: Still review Claude generated PR's.

4. **Extract and fetch the JIRA ticket.**

   Search for a JIRA ticket identifier using these sources in order:
   1. PR body/title (if PR exists): look for full URLs matching `airspacetechnologies.atlassian.net/browse/(AT|RR|INT|VAN)-\d+`
   2. PR body/title (if PR exists): look for standalone ticket patterns matching `(AT|RR|INT|VAN)-\d+`
   3. Branch name: look for a prefix matching `^(AT|RR|INT|VAN)-\d+`

   If a PR exists, use `gh pr view <PR> --json title,body,headRefName` to get these fields. Otherwise, use `git branch --show-current` for the branch name.

   - If a ticket ID is found, fetch it using the Atlassian MCP tools. This is a two-step process:
     1. First call `mcp__atlassian__getAccessibleAtlassianResources` to get the `cloudId` for the Atlassian site.
     2. Then call `mcp__atlassian__getJiraIssue` with that `cloudId`, the ticket ID as `issueIdOrKey` (e.g. `AT-22950`), `expand` set to `"renderedFields"`, and `fields` set to `["summary", "description", "status", "comment", "customfield_10400"]`. Note: `customfield_10400` is the "Acceptance Criteria" field in Airspace's JIRA.

     You MUST use these MCP tools — do NOT use WebFetch, Fetch, curl, or any other HTTP method to access JIRA. The MCP tools handle authentication automatically.

     Extract and store:
     - **Summary** (title)
     - **Description** (full description text)
     - **Acceptance Criteria** (from `customfield_10400`)
     - **Status**
     - **Comments** (may contain important context, requirements clarifications, or scope changes)
   - If no ticket ID is found, use `AskUserQuestion` to ask the user:
     - Provide the JIRA ticket ID (e.g. `AT-12345`)
     - Or describe the feature/requirements in free text

   Store the JIRA context (or user-provided description) for use in later steps.

5. Launch a haiku agent to return a list of file paths (not their contents) for all relevant CLAUDE.md files including:
   - The root CLAUDE.md file, if it exists
   - Any CLAUDE.md files in directories containing files modified by the pull request

6. Launch a sonnet agent to view the changes and return a summary. If a PR exists, use the PR diff. If no PR, use `git diff main...HEAD`.

7. Launch 5 agents in parallel to independently review the changes. Each agent should return the list of issues, where each issue includes a description and the reason it was flagged. The agents should do the following:

   **All agents receive:**
   - The JIRA context (summary, description, acceptance criteria) from step 4
   - The PR title and description (or branch name if no PR)
   - Any user-provided context from the arguments
   - If this is a re-review: the list of previous issues from step 2, so agents can check whether they are still present

   Agents 1 + 2: CLAUDE.md compliance sonnet agents
   Audit changes for CLAUDE.md compliance in parallel. Note: When evaluating CLAUDE.md compliance for a file, you should only consider CLAUDE.md files that share a file path with the file or parents.

   Agent 3: Opus bug agent (parallel subagent with agents 4 and 5)
   Scan for obvious bugs. Focus only on the diff itself without reading extra context. Flag only significant bugs; ignore nitpicks and likely false positives. Do not flag issues that you cannot validate without looking at context outside of the git diff.

   Agent 4: Opus bug agent (parallel subagent with agents 3 and 5)
   Look for problems that exist in the introduced code. This could be security issues, incorrect logic, etc. Only look for issues that fall within the changed code.

   Agent 5: Opus requirements fulfillment agent (parallel subagent with agents 3 and 4)
   Compare the PR changes against the JIRA ticket requirements and acceptance criteria from step 4. Flag:
   - Acceptance criteria that are not addressed by the code changes
   - Code that contradicts or only partially implements a requirement
   - Missing edge cases explicitly called out in the ticket
   Do NOT flag: requirements that are clearly out of scope for this PR (e.g. tracked in separate tickets), or acceptance criteria that cannot be verified from code alone (e.g. "verify in staging").

   **CRITICAL: We only want HIGH SIGNAL issues.** Flag issues where:
   - The code will fail to compile or parse (syntax errors, type errors, missing imports, unresolved references)
   - The code will definitely produce wrong results regardless of inputs (clear logic errors)
   - Clear, unambiguous CLAUDE.md violations where you can quote the exact rule being broken
   - Clear gaps between JIRA acceptance criteria and the implementation (Agent 5 only)

   Do NOT flag:
   - Code style or quality concerns
   - Potential issues that depend on specific inputs or state
   - Subjective suggestions or improvements

   If you are not certain an issue is real, do not flag it. False positives erode trust and waste reviewer time.

8. For each issue found in the previous step by agents 3, 4, and 5, launch parallel subagents to validate the issue. These subagents should get the PR title and description, the JIRA context, and a description of the issue. The agent's job is to review the issue to validate that the stated issue is truly an issue with high confidence. Use Opus subagents for bugs, logic issues, and requirements gaps, and sonnet agents for CLAUDE.md violations.

9. Filter out any issues that were not validated in step 8. This step will give us our list of high signal issues for our review.

   **Re-review reconciliation:** If this is a re-review (step 2 found prior feedback), compare the current issues against the previous issues:
   - **Fixed**: A previous issue that no longer appears in the current review. Mark it as resolved.
   - **Still open**: A previous issue that still exists in the current code. Keep it but note it was flagged before.
   - **New**: An issue that wasn't in the previous review. Flag it as new.

   Do NOT re-post or re-flag issues that have been fixed.

10. **Output the review based on the output mode:**

    **If terminal mode (default, no flags):**
    Output a summary of the review findings to the terminal:
    - If this is a re-review, start with a status summary: `X of Y previous issues fixed. Z new issues found.`
    - If issues remain or are new, list each with a brief description, the file/line, and the reason it was flagged.
    - If no issues, state: "No issues found. Checked for bugs, CLAUDE.md compliance, and JIRA requirements fulfillment."

    **If `--markdown` was provided:**
    Write a structured review to a `.md` file in the repo root. Use `review-PR-<number>.md` if a PR exists, or `review-<branch name>.md` otherwise. If a previous review file exists, **overwrite it** with the updated review. Format:

    ```markdown
    # Code Review: PR #<number> — <PR title>

    **JIRA Ticket:** <ticket ID> — <ticket summary>
    **Branch:** <branch name>
    **Reviewed:** <date>
    **Review round:** <1st review | Re-review>

    ## Summary
    <Brief summary of the PR changes>

    ## Previous Issues Status
    (Only include this section on re-reviews)
    - [FIXED] <issue description>
    - [STILL OPEN] <issue description>

    ## Issues Found

    ### Issue 1: <title>
    - **File:** `<path>:<line>`
    - **Category:** <Bug | CLAUDE.md Violation | Requirements Gap>
    - **Status:** <New | Still Open>
    - **Description:** <description>
    - **Suggestion:** <fix suggestion if applicable>

    (repeat for each issue, or "No issues found" section)

    ## Requirements Coverage
    <Summary of how well the PR addresses the JIRA acceptance criteria>
    ```

    **If `--comment` was provided:**
    - If this is a re-review, first post a summary comment using `gh pr comment` noting which previous issues were fixed and which remain.
    - If no issues were found (all fixed, none new), post a summary comment:

    ---

    ## Code review

    No issues found. Checked for bugs, CLAUDE.md compliance, and JIRA requirements fulfillment.

    **JIRA:** <ticket ID> — <ticket summary>

    ---

    - If issues remain or are new, continue to step 11.

11. (Only for `--comment` mode) Create a list of all comments that you plan on leaving. Do not re-comment on issues that were already commented on in a previous review and are still open — only comment on **new** issues. Do not post this list anywhere.

12. (Only for `--comment` mode) Post inline comments for each **new** issue using `mcp__github_inline_comment__create_inline_comment`. For each comment:
    - Provide a brief description of the issue
    - If the issue is a requirements gap, reference the JIRA ticket and specific acceptance criterion
    - For small, self-contained fixes, include a committable suggestion block
    - For larger fixes (6+ lines, structural changes, or changes spanning multiple locations), describe the issue and suggested fix without a suggestion block
    - Never post a committable suggestion UNLESS committing the suggestion fixes the issue entirely. If follow up steps are required, do not leave a committable suggestion.

    **IMPORTANT: Only post ONE comment per unique issue. Do not post duplicate comments.**

Use this list when evaluating issues in Steps 7 and 8 (these are false positives, do NOT flag):

- Pre-existing issues
- Something that appears to be a bug but is actually correct
- Pedantic nitpicks that a senior engineer would not flag
- Issues that a linter will catch (do not run the linter to verify)
- General code quality concerns (e.g., lack of test coverage, general security issues) unless explicitly required in CLAUDE.md
- Issues mentioned in CLAUDE.md but explicitly silenced in the code (e.g., via a lint ignore comment)

Notes:

- Use gh CLI to interact with GitHub (e.g., fetch pull requests, create comments). Do not use web fetch.
- Create a todo list before starting.
- You must cite and link each issue in inline comments (e.g., if referring to a CLAUDE.md, include a link to it).
- If no issues are found and `--comment` mode was specified, post a comment with the format shown in step 10.
- When linking to code in inline comments, follow the following format precisely, otherwise the Markdown preview won't render correctly: https://github.com/anthropics/claude-code/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
  - Requires full git sha
  - You must provide the full sha. Commands like `https://github.com/owner/repo/blob/$(git rev-parse HEAD)/foo/bar` will not work, since your comment will be directly rendered in Markdown.
  - Repo name must match the repo you're code reviewing
  - # sign after the file name
  - Line range format is L[start]-L[end]
  - Provide at least 1 line of context before and after, centered on the line you are commenting about (eg. if you are commenting about lines 5-6, you should link to `L4-7`)
