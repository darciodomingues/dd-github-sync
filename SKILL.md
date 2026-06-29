---
name: dd-github-sync
description: Super-powered GitHub operations skill that automatically chooses between MCP tools and gh CLI for optimal token usage, consistency, and reliability. USE THIS SKILL whenever interacting with GitHub (repos, issues, PRs, files, branches, search, releases, actions, etc.) — it picks the most efficient approach with production-grade patterns.
version: 2.0.0
---

# dd-github-sync — Super Skill for GitHub Operations

A production-grade wrapper around GitHub operations that intelligently routes between **MCP tools** (`mcp__plugin_github_github__*`) and **gh CLI** (`Bash` with `gh`) based on operation type, data volume, and context. Includes reusable patterns, error handling, rate limiting, and common workflow recipes.

---

## 🎯 Core Philosophy

**MCP for point operations (typed, structured, low-token) · gh CLI for bulk/search/streaming (filter early, paginate smart) · Hybrid for complex workflows**

| Dimension | MCP Tools | gh CLI |
|-----------|-----------|--------|
| **Best for** | Single resource CRUD, typed responses | Lists >20, search, pipelines, streaming |
| **Token cost** | Low (structured JSON) | Low *if* `--json` + `--jq` used correctly |
| **Reliability** | Auto-retry, typed errors | Manual retry, exit codes |
| **Flexibility** | Fixed schema | Full API surface via `gh api` |

---

## 🧠 Decision Algorithm (Enhanced)

```python
def choose_tool(operation: str, estimated_count: int, context: dict) -> ToolChoice:
    # Point operations → always MCP
    if operation in POINT_OPS:
        return MCP(tool=POINT_OP_MAP[operation])
    
    # Bulk/Search operations
    if operation in BULK_OPS:
        if estimated_count > 50:
            return GH_CLI(pattern="streaming", jq_filter=context.get("jq"))
        if estimated_count > 20:
            return GH_CLI(pattern="batched", jq_filter=context.get("jq"))
        return MCP(tool=BULK_OP_MAP[operation])  # small lists: MCP simpler
    
    # Complex workflows → hybrid
    if operation in WORKFLOW_OPS:
        return HYBRID(steps=context["steps"])
    
    # Unknown → default to gh CLI with safety limits
    return GH_CLI(pattern="safe_default", limit=100)
```

### Operation Classification

| Category | Operations | Default Tool |
|----------|------------|--------------|
| **POINT_OPS** | `get_repo`, `get_file`, `create_branch`, `create_pr`, `merge_pr`, `create_issue`, `update_issue`, `add_review`, `get_pr_diff`, `get_pr_files`, `create_release`, `get_release` | **MCP** |
| **BULK_OPS** | `list_issues`, `list_prs`, `list_repos`, `list_branches`, `list_commits`, `list_releases`, `list_workflows`, `list_runs` | **gh CLI** (if >20) / **MCP** (if ≤20) |
| **SEARCH_OPS** | `search_code`, `search_commits`, `search_issues`, `search_repos`, `search_users` | **gh CLI** (always) |
| **WORKFLOW_OPS** | `create_pr_from_issue`, `sync_fork`, `bulk_label_issues`, `auto_merge_when_ready`, `cleanup_merged_branches` | **HYBRID** |

---

## 🛠 gh CLI Patterns Library (Token-Optimized)

### Golden Rule: **Always `--json` + `--jq` filter BEFORE context**

```bash
# ❌ BAD - full objects flood context
gh pr list --repo owner/repo --state open

# ✅ GOOD - only needed fields, filtered early
gh pr list --repo owner/repo --state open --limit 100 \
  --json number,title,headRefName,author,reviewDecision,isDraft \
  --jq '.[] | {number, title, branch: .headRefName, author: .author.login, draft: .isDraft, review: .reviewDecision}'
```

### Reusable Field Sets (copy-paste)

```bash
# Minimal PR fields (90% of use cases)
PR_FIELDS="number,title,headRefName,baseRefName,author,state,isDraft,reviewDecision,createdAt,updatedAt"
PR_JQ='.[] | {number, title, head: .headRefName, base: .baseRefName, author: .author.login, state, draft: .isDraft, review: .reviewDecision}'

# Minimal Issue fields
ISSUE_FIELDS="number,title,state,author,labels,assignees,createdAt,updatedAt"
ISSUE_JQ='.[] | {number, title, state, author: .author.login, labels: .labels[].name, assignees: .assignees[].login}'

# Minimal Repo fields
REPO_FIELDS="name,description,private,updatedAt,stargazerCount,forkCount,primaryLanguage"
REPO_JQ='.[] | {name, description, private, updated: .updatedAt, stars: .stargazerCount, forks: .forkCount, language: .primaryLanguage?.name}'

# Minimal Commit fields
COMMIT_FIELDS="sha,message,author,committer,parents"
COMMIT_JQ='.[] | {sha: .sha[0:7], message: .commit.message | split("\n")[0], author: .commit.author.name, date: .commit.author.date}'
```

### Search Patterns (always use `gh api` for search)

```bash
# Code search - only paths + context lines
gh api search/code -f q="repo:owner/repo TODO" --per-page 50 \
  --jq '.items[] | {path: .path, repo: .repository.full_name, sha: .sha, match: .text_matches[0].fragment}'

# Commit search
gh api search/commits -f q="repo:owner/repo fix bug" --per-page 50 \
  --jq '.items[] | {sha: .sha[0:7], message: .commit.message | split("\n")[0], author: .commit.author.name, date: .commit.author.date}'

# Issue/PR search with filters
gh api search/issues -f q="repo:owner/repo is:pr is:open label:bug" --per-page 100 \
  --jq '.items[] | {number, title, state, author: .user.login, labels: .labels[].name}'
```

### Pipeline Patterns (list → filter → act)

```bash
# Pattern 1: Filter PRs needing review, then fetch details
gh pr list --repo o/r --state open --json number,reviewDecision,headRefName \
  --jq '.[] | select(.reviewDecision=="REVIEW_REQUIRED") | .number' | \
  xargs -I {} gh pr view {} --repo o/r --json files,title,body,reviews

# Pattern 2: Find stale branches, delete them
gh api repos/o/r/branches --paginate --jq '.[] | select(.commit.commit.author.date < "2024-01-01") | .name' | \
  xargs -I {} gh api -X DELETE repos/o/r/git/refs/heads/{}

# Pattern 3: Bulk label issues
gh issue list --repo o/r --state open --label "bug" --json number \
  --jq '.[].number' | xargs -I {} gh issue edit {} --repo o/r --add-label "priority:high"

# Pattern 4: Get all workflow runs for a PR
gh api repos/o/r/actions/runs --paginate -f head_sha=$(gh pr view 123 --repo o/r --json headRefOid --jq .headRefOid) \
  --jq '.workflow_runs[] | {name: .name, status: .conclusion, url: .html_url}'
```

---

## 🔄 Hybrid Workflow Recipes (Common Multi-Step Operations)

### Recipe 1: Create PR from Issue (with branch, commit, PR)
```python
# Step 1: MCP - Get issue details
issue = mcp.issue_read.get(owner, repo, issue_number)

# Step 2: MCP - Create branch from base
branch = f"issue-{issue_number}-{slugify(issue.title)}"
mcp.create_branch(owner, repo, branch, base="main")

# Step 3: gh CLI - Create commit (or MCP push_files)
# ... user makes changes ...

# Step 4: MCP - Create PR
pr = mcp.create_pull_request(owner, repo, title=issue.title, head=branch, base="main", body=f"Closes #{issue_number}")

# Step 5: MCP - Link PR to issue (auto via "Closes #")
```

### Recipe 2: Auto-merge when checks pass
```bash
# Poll until all checks pass, then merge
while true; do
  STATUS=$(gh pr view $PR --repo o/r --json statusCheckRollup --jq '.statusCheckRollup[] | select(.conclusion=="FAILURE") | .context')
  [ -z "$STATUS" ] && break
  sleep 30
done
gh pr merge $PR --repo o/r --squash --delete-branch
```

### Recipe 3: Sync fork with upstream
```bash
# Add upstream if needed
gh repo sync owner/fork --source upstream/repo --branch main
# Or manual:
git remote add upstream https://github.com/upstream/repo.git
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

### Recipe 4: Bulk close stale issues
```bash
gh issue list --repo o/r --state open --json number,updatedAt \
  --jq '.[] | select(.updatedAt < "2024-01-01") | .number' | \
  xargs -I {} gh issue close {} --repo o/r --reason "stale"
```

### Recipe 5: Get all changed files in a PR with diff stats
```bash
gh pr view $PR --repo o/r --json files --jq '.files[] | {path: .path, additions: .additions, deletions: .deletions, status: .status}'
```

---

## ⚙️ Configuration & Defaults

```yaml
# .claude/skills/dd-github-sync/config.yaml (optional)
defaults:
  per_page: 100              # gh CLI pagination
  max_pages: 10              # safety cap
  timeout_seconds: 30        # gh CLI timeout
  retry_attempts: 3          # gh CLI retries
  retry_backoff: 2           # exponential base
  
token_optimization:
  always_json_jq: true
  max_items_in_context: 50   # truncate if exceeded
  summarize_large_lists: true
  
fallback:
  mcp_on_gh_failure: true
  gh_on_mcp_failure: true
  
aliases:
  me: "{{github_user}}"
  my_org: "{{github_org}}"
```

---

## 🛡 Error Handling & Resilience

### gh CLI Wrapper Function (use in Bash calls)
```bash
gh_retry() {
  local max_attempts=3
  local attempt=1
  local backoff=2
  
  while [ $attempt -le $max_attempts ]; do
    if output=$(gh "$@" 2>&1); then
      echo "$output"
      return 0
    fi
    
    local exit_code=$?
    if echo "$output" | grep -q "rate limit\|secondary rate limit"; then
      sleep $((backoff * attempt))
      attempt=$((attempt + 1))
      continue
    fi
    
    # Non-rate-limit error - fail fast
    echo "gh command failed (attempt $attempt): $output" >&2
    return $exit_code
  done
  
  echo "gh command failed after $max_attempts attempts" >&2
  return 1
}

# Usage: gh_retry pr list --repo o/r --json number,title --jq '.[] | {number, title}'
```

### MCP Fallback Pattern
```python
try:
    result = mcp.pull_request_read.get_diff(owner, repo, pr_number)
except MCPError as e:
    if "rate limit" in str(e).lower():
        # Fall back to gh CLI
        result = bash(f"gh pr view {pr_number} --repo {owner}/{repo} --json diff --jq .diff")
    else:
        raise
```

---

## 📋 Quick Reference Card

| Task | Command |
|------|---------|
| **My PRs needing review** | `gh pr list --author @me --state open --json number,title,reviewDecision --jq '.[] | select(.reviewDecision=="REVIEW_REQUIRED")'` |
| **Open PRs in repo** | `gh pr list --repo o/r --state open --json number,title,headRefName,author --jq PR_JQ` |
| **Search code** | `gh api search/code -f q="repo:o/r fn" --jq '.items[] | {path, sha}'` |
| **Create PR** | `mcp.create_pull_request(owner, repo, title, head, base, body)` |
| **Merge PR** | `mcp.merge_pull_request(owner, repo, pr_number, method="squash")` |
| **Get file content** | `mcp.get_file_contents(owner, repo, path, ref="main")` |
| **Push file** | `mcp.create_or_update_file(owner, repo, path, content, message, branch)` |
| **List workflows** | `gh api repos/o/r/actions/workflows --jq '.workflows[] | {id, name, path, state}'` |
| **Trigger workflow** | `gh api -X POST repos/o/r/actions/workflows/id/dispatches -f ref=main -f inputs='{}'` |
| **Get run logs** | `gh run view RUN_ID --repo o/r --log` |

---

## 🔌 Integration with Other Skills

- **`claude-code-setup:claude-automation-recommender`** → Add gh auth to settings
- **`frontend-design:frontend-design`** → PR preview deployments
- **`mcp-server-dev:build-mcp-server`** → GitHub webhook handlers
- **`superpowers:subagent-driven-development`** → Parallel PR reviews

---

## ✅ Validation Checklist (run before complex ops)

- [ ] Estimated result count < 50? → Consider MCP
- [ ] Using `--json` + `--jq` on every gh list/search?
- [ ] Pagination limited (`--limit`, `--per-page`)?
- [ ] Rate limit handling in place?
- [ ] Fallback defined for critical operations?
- [ ] Only needed fields selected?
- [ ] Large outputs summarized/truncated?

---

## 🎓 Skill Evolution Notes

This skill embodies **token-aware GitHub operations**. Every pattern here was chosen because it:
1. Minimizes tokens entering context (filter early!)
2. Uses typed MCP where structure matters
3. Uses streaming gh CLI where volume matters
4. Provides fallback for resilience
5. Documents the *why* for future maintenance

**When in doubt**: Estimate count → if >20, gh CLI with `--json` + `--jq`; else MCP.