---
name: appwrite-review-threads
description: Applies when a pull request has a summary comment from Appwrite Review (its body contains <!-- appwrite-hansi:summary) or review comments from the Appwrite GitHub App, including when fixing findings, replying, or otherwise editing the pull request. Do not resolve Appwrite Review threads. Appwrite Review clears them after a fix or a dismissal.
---

# Appwrite Review threads

When a pull request has a summary comment from Appwrite Review (its body contains `<!-- appwrite-hansi:summary`) or review comments from the Appwrite GitHub App:

- Do not call `resolveReviewThread`, and do not resolve those threads in the GitHub UI. A resolved thread stays open in Appwrite Review's summary, so the tier stays capped.
- Fix the code, or reply on that thread. Appwrite Review resolves the thread when a later review sees the fix, or when it dismisses the finding after the reply.
- `appwrite-review-status` reads the review. `appwrite-review-learn` saves a lasting rule. `appwrite-review-setup` changes `.hansi.json`. `appwrite-review-loop` fixes until Tier S.
