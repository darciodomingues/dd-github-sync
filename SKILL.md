---
name: dd-github-sync
description: Smart GitHub operations skill that automatically chooses between MCP tools and gh CLI for optimal token usage and consistency. USE THIS SKILL whenever the user wants to interact with GitHub (repos, issues, PRs, files, branches, search, etc.) — it will pick the most token-efficient approach automatically.
---

# dd-github-sync

A smart wrapper around GitHub operations that chooses between **MCP tools** (`mcp__plugin_github_github__*`) and **gh CLI** (`Bash` with `gh` commands) based on the operation type to minimize token usage while maintaining consistency.

## Core Philosophy

**MCP for point operations, gh CLI for bulk/search operations.**

| Use MCP (low token, structured) | Use gh CLI (token-efficient for bulk) |
|----------------------------------|----------------------------------------|
| Get single resource (issue, PR, file, repo) | List/search > 20 items (issues, PRs, repos, commits, code) |
| Create/update single resource (branch, PR, file, issue) | Search code, commits, issues, repos |
| Get diff/files for single PR | Pipeline multi-step ops (list → filter → act) |
| Submit review, merge PR | Bulk operations with jq filtering |

---

## Decision Algorithm

```
IF operation in {get_single, create_single, update_single, delete_single}:
    USE MCP
ELIF operation in {list, search} AND expected_count > 20:
    USE gh CLI with --json + --jq filtering
ELIF operation in {list, search} AND expected_count <= 20:
    USE MCP (simpler, structured output)
ELSE:
    USE gh CLI (safer default for unknown volume)
```

---

## MCP Tools Reference (use for point ops)

| Category | Tools |
|----------|-------|
| **Repos** | `get_file_contents`, `create_or_update_file`, `delete_file`, `push_files`, `create_repository`, `fork_repository` |
| **Issues** | `issue_read` (get, get_comments, get_labels), `issue_write` (create, update) |
| **PRs** | `pull_request_read` (get, get_diff, get_files, get_commits, get_review_comments, get_reviews, get_comments, get_status, get_check_runs), `create_pull_request`, `update_pull_request`, `merge_pull_request`, `update_pull_request_branch` |
| **Reviews** | `pull_request_review_write` (create, submit_pending, resolve_thread, unresolve_thread), `add_comment_to_pending_review`, `add_reply_to_pull_request_comment` |
| **Branches/Tags** | `create_branch`, `list_branches`, `list_tags`, `get_tag` |
| **Releases** | `get_latest_release`, `get_release_by_tag`, `list_releases` |
| **Search** | `search_code`, `search_commits`, `search_issues`, `search_pull_requests`, `search_repositories`, `search_users` |
| **Collaborators** | `list_repository_collaborators` |
| **Misc** | `get_me`, `get_commit`, `run_secret_scanning`, `request_copilot_review` |

---

## gh CLI Patterns (use for bulk/search)

### List with filtering (always use `--json` + `--jq`)
```bash
# Issues/PRs - only fetch needed fields
gh issue list --repo owner/repo --state open --limit 100 \
  --json number,title,state,author,labels,createdAt \
  --jq '.[] | {number, title, author: .author.login, labels: .labels[].name}'

gh pr list --repo owner/repo --state open --limit 50 \
  --json number,title,headRefName,author,reviewDecision \
  --jq '.[] | {number, title, branch: .headRefName, author: .author.login}'

# Repos
gh repo list owner --limit 100 --json name,description,private,updatedAt \
  --jq '.[] | {name, description, private, updated: .updatedAt}'

# Commits
gh api repos/owner/repo/commits --paginate \
  --jq '.[] | {sha: .sha, message: .commit.message, author: .commit.author.name, date: .commit.author.date}'
```

### Search (code, commits, issues, repos)
```bash
# Code search - only return paths
gh api search/code -f q="repo:owner/repo function_name" \
  --jq '.items[] | {path: .path, sha: .sha, repo: .repository.full_name}'

# Commit search
gh api search/commits -f q="repo:owner/repo fix bug" \
  --jq '.items[] | {sha: .sha, message: .commit.message, author: .commit.author.name}'

# Issue/PR search
gh api search/issues -f q="repo:owner/repo is:pr is:open" \
  --jq '.items[] | {number, title, state, author: .user.login}'
```

### Pipeline pattern (list → filter → act)
```bash
# Get PR numbers needing review, then fetch details
gh pr list --repo owner/repo --state open --json number,reviewDecision \
  --jq '.[] | select(.reviewDecision=="REVIEW_REQUIRED") | .number' | \
  xargs -I {} gh pr view {} --repo owner/repo --json files,title,body
```

---

## Skill Usage Rules

1. **ALWAYS** estimate result count before choosing tool
2. **ALWAYS** use `--json` + `--jq` with gh CLI to minimize output tokens
3. **NEVER** use `gh` without `--json` for list/search operations
4. **PREFER** MCP for create/update/delete single resources (structured, typed)
5. **PREFER** gh for any operation that might return > 50 items
6. **COMBINE** when needed: MCP for auth/setup, gh for bulk, MCP for final action

---

## Example Decisions

| User Request | Decision | Reason |
|--------------|----------|--------|
| "Get issue #123" | MCP `issue_read` | Single resource |
| "List all open issues" | gh CLI | Unknown count, likely > 20 |
| "Create PR from branch X" | MCP `create_pull_request` | Single create |
| "Search for 'TODO' in codebase" | gh CLI `search/code` | Search returns many |
| "Get PR #45 diff" | MCP `pull_request_read` get_diff | Single resource |
| "List all repos in org" | gh CLI `repo list` | Could be hundreds |
| "Merge PR #10" | MCP `merge_pull_request` | Single action |
| "Find all repos with topic:python" | gh CLI `api search/repositories` | Search + bulk |

---

## Error Handling

- MCP: Returns structured errors, auto-retries on rate limit
- gh CLI: Check exit code, stderr; implement retry with exponential backoff for rate limits
- On MCP failure for bulk ops → fall back to gh CLI
- On gh CLI failure for point ops → fall back to MCP

---

## Token Optimization Tips

1. **Filter early**: Use `--jq` to select only needed fields BEFORE data reaches context
2. **Paginate wisely**: `--limit N` on gh, `perPage` on MCP
3. **Avoid full objects**: Never fetch full PR/issue objects when only title/number needed
4. **Batch when possible**: gh CLI pipes multiple API calls in one tool invocation