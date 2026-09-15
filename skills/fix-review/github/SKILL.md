---
name: fix-review-github
description: >-
  GitHub CLI commands for fetching PR metadata and diffs in the fix agent.
  Use gh to view PR state, diff, and comments.
---

# Fix Review — GitHub CLI

Use the `gh` CLI to fetch PR data for the fix agent. The environment
provides `GH_TOKEN` for authentication.

## PR Metadata

```bash
# View PR with full details
gh pr view "${PR_NUMBER}" --json number,title,body,headRefName,baseRefName,state,files,labels

# View PR state only
gh pr view "${PR_NUMBER}" --json state --jq '.state'
```

## PR Diff

```bash
# Fetch the current diff
gh pr diff "${PR_NUMBER}"
```

## Review findings fallback

The review bot posts findings as an issue comment marked
`<!-- fullsend:review-agent -->`, not as a PR review body. COMMENT
reviews contain only a pointer to that comment. When
`/sandbox/workspace/review-body.txt` is empty or whitespace-only, fetch
the latest matching issue comment. Use jq `last` (not `tail -1`) because
comment bodies contain newlines.

```bash
REVIEW_BODY_FILE="/sandbox/workspace/review-body.txt"
REVIEW_COMMENT=$(gh api --paginate --slurp "repos/${REPO_FULL_NAME}/issues/${PR_NUMBER}/comments" \
  | jq -r 'add // [] | [.[] | select(.user.login | endswith("-review[bot]")) | select(.body | contains("<!-- fullsend:review-agent -->"))] | last | .body // empty')
if [ -n "${REVIEW_COMMENT}" ]; then
  echo "::notice::Recovered review findings from issue comment API fallback"
  printf '%s\n' "${REVIEW_COMMENT}" > "${REVIEW_BODY_FILE}"
else
  echo "::error::No review body found at ${REVIEW_BODY_FILE} and API fallback found no review comment"
fi
cat "${REVIEW_BODY_FILE}"
```
