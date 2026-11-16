---
name: appwrite-hansi-loop
description: Iteratively improves a GitHub pull request until Hansi (Appwrite Review) grades it Tier S (ready to merge) with zero unresolved comments. Triggers a review, fixes actionable findings, pushes, and repeats. Use when the user wants to clear Hansi's or Appwrite Review's comments, reach Tier S, loop on review comments, or run hansi-loop.
---

# Hansi loop

Hansi is the reviewer behind Appwrite Review. Fix a pull request until Hansi grades it **Tier S** (ready to merge) and leaves no unresolved comments.

Each review counts toward the Appwrite project's review usage. Do not request a new review when the current head already has a finished Hansi review and you have not pushed anything since.

The command blocks are examples, written for git and the GitHub CLI. Run whatever this environment actually provides to do the same job, including commands the examples do not list. If `gh` or `git` is missing, or a command fails because of how this machine authenticates, try another client before stopping.

## Inputs

- **Pull request number** (optional). If omitted, use the pull request for the current branch.

## Instructions

### 1. Identify the pull request

Set `PR_NUMBER` when the user gave one. Leave it empty to use the current branch.

```bash
if [ -n "$PR_NUMBER" ]; then
  gh pr view "$PR_NUMBER" --json number,headRefName,headRefOid,url -q '{number: .number, branch: .headRefName, sha: .headRefOid, url: .url}'
else
  gh pr view --json number,headRefName,headRefOid,url -q '{number: .number, branch: .headRefName, sha: .headRefOid, url: .url}'
fi
```

Use the `number` from that JSON as `<PR_NUMBER>` in every later command. Check out `branch` if you are not already on it. Stop if there is no open pull request.

### 2. Find the summary and the mention handle

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

When several projects review the repository, ask the user which one to loop on, then use only that summary's `<REPOSITORY_ID>`. A push or a mention starts a review in every project; that is expected.

If there is no summary yet, read the slug from the `slug` field of a `Hansi` check on the pull request (drop the `external_id` filter). If you still cannot find a handle, ask the user. Do not guess, and do not mention a different bot.

### 3. Loop

Repeat the cycle below. **Stop after 5 iterations.**

#### A. Make sure a review of the current head is running

Push commits that are not on the remote yet:

```bash
git status --short --branch
git push
```

A push reviews the new commits on its own when automatic reviews are enabled (the default). Look at the `Hansi` check on the **current** head SHA, not an older one:

```bash
HEAD_SHA=$(gh pr view <PR_NUMBER> --json headRefOid -q .headRefOid)
gh api "repos/{owner}/{repo}/commits/$HEAD_SHA/check-runs?check_name=Hansi" \
  --jq '[.check_runs[] | select(.name == "Hansi" and .external_id == "<REPOSITORY_ID>")] | sort_by(.started_at) | last | {status, conclusion, started_at, title: .output.title, summary: .output.summary}'
```

A completed check for this head is the review to read. Do not request another one.

If this head has no `Hansi` check, wait about 30 seconds and look again. The webhook needs a moment after a push. Comment only if it is still missing. Skip the comment when `status` is `queued` or `in_progress`.

```bash
gh pr comment <PR_NUMBER> --body "@<slug> review"
```

The comment must be plain text. A mention inside a code span or a quote does not start a review.

Poll until that check completes. Reviews read the repository, so allow about 15 minutes (90 attempts, 10 seconds apart):

```bash
ATTEMPTS=0
MAX_ATTEMPTS=90
while true; do
  ATTEMPTS=$((ATTEMPTS + 1))
  if [ "$ATTEMPTS" -gt "$MAX_ATTEMPTS" ]; then
    echo "Timed out waiting for the Hansi check." >&2
    exit 1
  fi
  CHECK=$(gh api "repos/{owner}/{repo}/commits/$HEAD_SHA/check-runs?check_name=Hansi" \
    --jq '[.check_runs[] | select(.name == "Hansi" and .external_id == "<REPOSITORY_ID>")] | sort_by(.started_at) | last')
  STATUS=$(printf '%s' "$CHECK" | jq -r '.status // empty')
  if [ -z "$STATUS" ]; then
    echo "Waiting for the Hansi check to appear..."
    sleep 10
    continue
  fi
  if [ "$STATUS" = "completed" ]; then
    printf '%s' "$CHECK" | jq -r '"Hansi check completed: \(.conclusion) — \(.output.title)"'
    break
  fi
  echo "Waiting for Hansi... ($STATUS)"
  sleep 10
done
```

Stop the loop, and do not use an older review, when:

- polling times out
- `conclusion` is `skipped` (drafts, disabled reviews, or a base branch Hansi does not review)
- the check title is `Review failed`

`failure` means Hansi requested changes. `neutral` means it commented. Both are finished reviews: keep going and read the comments.

#### B. Read the review

Fetch the summary comment again (the command in step 2). Hansi edits that same comment in place. Parse:

- **Tier**, from the heading `Tier S` through `Tier F`. **S** means ready to merge. Open findings cap it: a minor finding means at most **A**, a major one at most **B**, a critical one at most **D**. Informational notes do not lower it.
- **Verdict table**: new comments, fixed, and still open.
- **Findings table** of comments posted on this review. A row is a title plus a file and line.
- **Still open from earlier reviews**, inside the `⏳ Still open from earlier reviews` details block.
- **Filtered out**, inside the `🔇 Filtered out` details block. These were considered and not posted. Include the reason.
- **Approval withheld**, a `> [!NOTE]` block. Hansi will not approve in that case. A note about an open bug finding clears once that finding is fixed or dismissed. A note about write access or an incomplete diff does not.

Then read the threads.

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

#### C. Stop when the pull request is clean

Stop when **all** of these are true:

- The tier in the current summary is **S**
- Hansi has **no unresolved threads**, and the summary has no findings still open
- You are not waiting on a review of commits you just pushed

Then, if the latest `Hansi` check on this head is `failure` (the pull request is still in "changes requested"), request **one** `@<slug> review` and wait for it, so Hansi can submit the approval. Do not repeat that extra review. If the follow-up still does not approve, stop and report the summary note. A withheld approval or `reviews.approve: false` in `.hansi.json` will not become an approval.

Also stop when the fifth iteration finishes. Report whatever is left.

#### D. Fix what Hansi asked for

For each unresolved Hansi finding:

1. Read the file, the diff hunk, and the comment.
2. Fix it in the code when it describes a real bug. Apply a suggestion block only when it actually fixes that bug.
3. When the comment does not apply, reply on **that thread** with a short reason. A reply to a finding is enough; you do not need to mention Hansi again. Hansi answers, and dismisses the finding when it agrees.

```bash
gh api "repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments/<COMMENT_ID>/replies" -f body='...'
```

Do **not** call `resolveReviewThread` yourself. Resolving a thread in GitHub leaves the finding open in Hansi's summary, so the tier stays capped. Hansi resolves the thread when a later review sees the fix, or when it dismisses the finding after your reply.

#### E. Wait for replies before reviewing again

If you replied on any thread, wait until Hansi has answered (it reacts with 👀, then comments) or about 5 minutes have passed. A dismissal updates the summary comment. Read it again before you decide the iteration still needs code changes.

#### F. Commit and push code changes

If you changed files, commit only those files. Leave unrelated work unstaged.

```bash
git add path/to/file
git commit -m "Address Hansi review feedback (iteration N)"
git push
```

Go back to step **A**. The push starts the next review; do not also comment `@<slug> review` unless that check never appears.

If the only action was thread replies and nothing remains open, go back to step **C** without pushing an empty commit.

### 4. Report

After the loop, summarize:

```
hansi-loop complete.
  Pull request:  #123
  Iterations:    2
  Tier:          S
  Resolved:      4 comments
  Remaining:     0
```

When the loop stops early:

```
hansi-loop stopped after 5 iterations.
  Pull request:  #123
  Tier:          A
  Resolved:      6 comments
  Remaining:     1

Remaining issues:
  - src/auth.ts:45 — "Session cookie is not marked HttpOnly"
```

Include the pull request URL. If Hansi withheld approval, say so in the report even when the tier is **S** and nothing is left open.
