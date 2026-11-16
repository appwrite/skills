---
name: appwrite-hansi-threads
description: Applies when a pull request has a Hansi summary comment (its body contains <!-- appwrite-hansi:summary) or review comments from the Appwrite GitHub App, including when fixing findings, replying, or otherwise editing the pull request. Do not resolve Hansi review threads. Hansi clears them after a fix or a dismissal.
---

# Hansi review threads

Hansi is the reviewer behind Appwrite Review. When a pull request has a Hansi summary comment (its body contains `<!-- appwrite-hansi:summary`) or review comments from the Appwrite GitHub App:

- Do not call `resolveReviewThread`, and do not resolve those threads in the GitHub UI. A resolved thread stays open in Hansi's summary, so the tier stays capped.
- Fix the code, or reply on that thread. Hansi resolves the thread when a later review sees the fix, or when it dismisses the finding after the reply.
- `appwrite-hansi-status` reads the review. `appwrite-hansi-learn` saves a lasting rule. `appwrite-hansi-setup` changes `.hansi.json`. `appwrite-hansi-loop` fixes until Tier S.
