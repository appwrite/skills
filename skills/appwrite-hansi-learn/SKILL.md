---
name: appwrite-hansi-learn
description: Replies on a Hansi finding (Appwrite Review) with a lasting review rule so Hansi remembers it for future reviews of the repository. Use when the user says a finding should not be flagged, asks Hansi or Appwrite Review to remember a preference, wants a disagreement saved as a team rule, or runs hansi-learn.
---

# Hansi learn

Hansi is the reviewer behind Appwrite Review. Save a lasting review rule by replying on the Hansi finding it applies to. Hansi stores the rule as a learning of that Appwrite project's repository when a reply states a durable preference, then follows it on later reviews. Learnings are listed under Review in the Appwrite Console.

Do not request a review. Do not push. Do not call `resolveReviewThread`. Resolving a thread in GitHub leaves the finding open in Hansi's summary, so the tier stays capped. Hansi resolves the thread when it dismisses the finding.

The command blocks are examples, written for git and the GitHub CLI. Run whatever this environment actually provides to do the same job, including commands the examples do not list. If `gh` or `git` is missing, or a command fails because of how this machine authenticates, try another client before stopping.

## Inputs

- **Pull request number** (optional). If omitted, use the pull request for the current branch.
- **Finding** (optional). A comment id, a file and line, or a quote of the title. If omitted, use the unresolved Hansi finding the user is talking about. If several match, ask which one.
- **Rule** (optional). If the user did not state a lasting preference, ask once, then stop until they answer. Do not invent a rule.

## Instructions

### 1. Identify the pull request and the finding

```bash
if [ -n "$PR_NUMBER" ]; then
  gh pr view "$PR_NUMBER" --json number,url -q '{number: .number, url: .url}'
else
  gh pr view --json number,url -q '{number: .number, url: .url}'
fi
```

Use the `number` as `<PR_NUMBER>`. Stop if there is no open pull request.

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

The finding comment's `databaseId` is `<COMMENT_ID>`. A rule is saved for the project whose finding it replies to, so reply on a finding of the project the user means.

If the user did not identify a finding and more than one Hansi thread is unresolved, ask which one. Do not reply on every thread.

### 2. Write the rule

One imperative sentence, understandable without this thread. Name the path or situation it covers.

```
Do not flag missing error handling in tests.
```

A one-off ("ignore this once", "this line is fine today") is not a rule. Say so and stop without posting.

Show the sentence to the user when they did not write it themselves, and wait for a yes before posting.

### 3. Reply on that thread

The reply must be plain text on the finding. Do not mention `@<slug> review`. A review mention starts a new review.

```bash
gh api "repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments/<COMMENT_ID>/replies" \
  -f body='Do not flag missing error handling in tests. This is a lasting review rule for this repository.'
```

Lead with the rule. Say that it is a lasting review rule for this repository so Hansi treats it as a preference to remember, not a one-off.

### 4. Wait for Hansi

Hansi reacts with 👀, then comments. Poll the thread for about 5 minutes (30 attempts, 10 seconds apart). A new comment from `<slug>[bot]` after the reply is the answer.

Read that comment:

- Hansi agrees and says it will follow the rule: the preference is saved. Future reviews of this repository in that project use it. The summary comment updates when the finding is dismissed.
- Hansi disagrees, or treats the reply as a one-off: the rule was not saved. Quote the answer and stop. Do not post a second reply unless the user asks.

### 5. Report

```
hansi-learn
  Pull request:  #123
  Finding:       src/auth.test.ts:12 — "Missing error handling"
  Rule:          Do not flag missing error handling in tests.
  Saved:         yes
```

Include the pull request URL. When it was not saved, set `Saved` to `no` and quote Hansi's reply.
