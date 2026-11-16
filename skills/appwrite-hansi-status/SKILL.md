---
name: appwrite-hansi-status
description: Reads the current Hansi review (Appwrite Review) on a GitHub pull request and reports the tier, open findings, filtered-out comments, and whether approval was withheld. Does not request a new review or change the pull request. Use when the user wants to know what Hansi or Appwrite Review said, the current tier, or which comments are still open, or when they run hansi-status.
---

# Hansi review status

Hansi is the reviewer behind Appwrite Review. Read its review of a pull request and stop. Do not comment, push, commit, or resolve a thread. A new review counts toward the Appwrite project's review usage.

The command blocks are examples, written for git and the GitHub CLI. Run whatever this environment actually provides to do the same job, including commands the examples do not list. If `gh` or `git` is missing, or a command fails because of how this machine authenticates, try another client before stopping.

## Inputs

- **Pull request number** (optional). If omitted, use the pull request for the current branch.

## Instructions

### 1. Identify the pull request

Set `PR_NUMBER` when the user gave one. Leave it empty to use the current branch.

```bash
if [ -n "$PR_NUMBER" ]; then
  gh pr view "$PR_NUMBER" --json number,headRefName,headRefOid,url,baseRefName -q '{number: .number, branch: .headRefName, sha: .headRefOid, url: .url, base: .baseRefName}'
else
  gh pr view --json number,headRefName,headRefOid,url,baseRefName -q '{number: .number, branch: .headRefName, sha: .headRefOid, url: .url, base: .baseRefName}'
fi
```

Use the `number` from that JSON as `<PR_NUMBER>` and the `sha` as `<HEAD_SHA>`. Stop if there is no open pull request.

### 2. Read the summary

Hansi keeps one issue comment per pull request up to date and edits it in place. Its body starts with the marker `<!-- appwrite-hansi:summary:<REPOSITORY_ID> -->`, where `<REPOSITORY_ID>` is the Appwrite Review repository that wrote it.

```bash
gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  --jq '.[] | select(.body | contains("<!-- appwrite-hansi:summary")) | {id, updated_at, body}'
```

There is usually one. Several Appwrite projects can review the same repository, and each writes its own summary. When there are several and the user did not say which project they mean, work with each one separately and say which is which. Never mix findings from two summaries.

The footer looks like `Comment <code>@app-name review</code> to re-run`. That `@<slug>` is Hansi's mention handle on this repository; take it from inside the `<code>` tag. Review comments are authored by `<slug>[bot]`.

The `Hansi` check run of a summary is the one whose `external_id` is that summary's `<REPOSITORY_ID>`:

```bash
gh api "repos/{owner}/{repo}/commits/<HEAD_SHA>/check-runs?check_name=Hansi" \
  --jq '[.check_runs[] | select(.name == "Hansi" and .external_id == "<REPOSITORY_ID>")] | sort_by(.started_at) | last | {status, conclusion, head_sha, title: .output.title, summary: .output.summary, slug: .app.slug}'
```

A check run written before reviews were keyed by repository has no `external_id`. When no check matches, drop the `external_id` filter and use the latest `Hansi` check, but only when there is a single summary.

If there is no summary, say there is no Hansi review yet and stop. Do not mention the app and do not request a review.

Parse the body:

- **Tier**, from the heading `Tier S` through `Tier F`. **S** means ready to merge. Open findings cap it: a minor finding means at most **A**, a major one at most **B**, a critical one at most **D**. Informational notes do not lower it.
- **Verdict table**: new comments, fixed, and still open.
- **Findings table** of comments posted on this review. A row is a title plus a file and line.
- **Still open from earlier reviews**, inside the `⏳ Still open from earlier reviews` details block.
- **Filtered out**, inside the `🔇 Filtered out` details block. These were considered and not posted. Include the reason.
- **Approval withheld**, a `> [!NOTE]` block. Hansi will not approve in that case. A note about an open bug finding clears once that finding is fixed or dismissed. A note about write access or an incomplete diff does not.

### 3. Read the check on this head

Read the `Hansi` check of that summary on `<HEAD_SHA>` (the second command above) and report it exactly:

- `queued` or `in_progress`: a review of this head is still running. Do not wait for it.
- `completed` with `failure`: Hansi requested changes.
- `completed` with `neutral`: Hansi commented.
- `completed` with `success`: Hansi approved.
- `skipped`: drafts, disabled reviews, or a base branch Hansi does not review.
- Title `Review failed`: the review did not finish.
- No check on `<HEAD_SHA>`: the summary may be for an older commit. Say the review is stale and name the head it does not cover.

### 4. List unresolved Hansi threads

List unresolved review threads and keep the ones whose first comment is from `<slug>[bot]`:

```bash
gh api graphql -f query='
query($cursor: String) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          comments(first: 20) {
            nodes { databaseId body path author { login } }
          }
        }
      }
    }
  }
}'
```

Page with `endCursor` while `hasNextPage` is true. A finding comment starts with a bold title, then the explanation, sometimes a `suggestion` block, and a footer like `Major · bug`. When several projects review the repository, all of their findings come from the same bot account: match a thread to a summary by its title in that summary's findings tables.

Also read the latest Hansi review body. If GitHub rejected the inline comments, the findings are in that body instead of threads. Treat those as open too.

Do not call `resolveReviewThread`.

### 5. Report

```
hansi-status
  Pull request:  #123
  URL:           https://github.com/owner/repo/pull/123
  Head:          abc1234
  Check:         completed · failure
  Tier:          B
  Open:          2
  Filtered out:  1
  Approval withheld: no

Open findings:
  - src/auth.ts:45 — "Session cookie is not marked HttpOnly" (major, unresolved)

Filtered out:
  - "Rename this variable" — style
```

Quote titles from the summary. If the summary and the threads disagree, list both and say which source each line came from. If approval was withheld, quote the note even when the tier is **S**. When several projects review the repository, give one report per summary.
