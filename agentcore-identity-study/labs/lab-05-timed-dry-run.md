# Lab 5: Timed dry run (135 minutes)

Goal: practice the real format: a broken environment, no hints, a clock. Set a timer for 135 minutes. Do not open your notes for the first 60 minutes. After that you may use the cheat sheet only.

## Setup (not timed, 20 minutes)
Build a "broken" environment in a clean region or with a unique name prefix. Better: ask someone else to break it, or write down the faults, then wait a day so you forget the details.

Create:
- a JWT runtime (Cognito) with these faults: wrong `discoveryUrl` pool ID typo, `allowedAudience` set while using an access token, and `allowedClients` missing the real client
- a SigV4 runtime where the caller role has `InvokeAgentRuntime` on `runtime/*` and the user-ID Deny is missing
- two workload identities with no tags, and one role with `bedrock-agentcore:*` on `*`
- an OAuth2 or API key provider with a policy that lists only the provider ARN (three resources missing)
- no CloudWatch alarms, and observability disabled

## Tasks (the timed part)
Write down the start and finish time of each.

| # | Task | Objective | Target time | Actual |
|---|------|-----------|-------------|--------|
| 1 | Make the JWT runtime accept the valid token and reject other clients | 2 | 20 min | |
| 2 | Add a scope or custom claim rule and prove it works with two tokens | 2 | 15 min | |
| 3 | Fix the SigV4 path: least-privilege caller policy, Deny the user-ID action, optional resource policy | 2 | 20 min | |
| 4 | Create a workload identity for a self-hosted agent, tag it, and restrict access by tag | 1 | 25 min | |
| 5 | Rewrite the over-broad agent role and the provider policy to least privilege | 1 | 15 min | |
| 6 | Enable observability, create an alarm for token fetch failures and one CloudTrail based alert, then trigger both | 3 | 25 min | |
| 7 | Write down the evidence for each fix (status code before and after) | all | 10 min | |
| | Total | | 130 min | |

## Scoring
For each task: passed / partly / failed. Passed means you can show before and after evidence, such as a 401 turning into 200 or an AccessDenied appearing where it should.

## Debrief (15 minutes)
- Where did you lose the most time?
- Which error messages did you not recognize? Add them to `cheatsheet.md`.
- Which commands did you have to look up? Practice those three times.
- Which assumption in the notes was wrong? Fix the note.
- Repeat the failed tasks tomorrow. A second dry run is worth the time if you finished less than 80%.

## Exam tactics
- Read all tasks first. Do the cheap ones first.
- Test after each change. Do not stack three changes before testing.
- Decode the token before you touch the authorizer.
- Use explicit ARNs in IAM statements. Watch for the four-resource rule.
- Note the lab environment's region and account at the start.

## Cleanup checklist
- [ ] Delete runtimes, workload identities, credential providers
- [ ] Delete Cognito pool, roles, policies
- [ ] Delete alarms, metric filters, and any trail you made
- [ ] Check billing/Cost Explorer next day for leftovers
