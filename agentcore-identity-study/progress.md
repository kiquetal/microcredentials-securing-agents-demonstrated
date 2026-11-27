# Progress tracker

The lab coach agent reads this file at the start of every session and updates it at the end. You can edit it by hand too. The README progress table is the short summary. This file holds the detail.

Rules for entries: no tokens, secrets or real account IDs (use `111122223333`). Dates are `YYYY-MM-DD`.

## Next action
**Block 1: read notes 00, 01 and 03, then start Lab 1.** (Agent: replace this line at the end of every session.)

## Blocks
| Block | Content | Status | Started | Finished | Time spent |
|-------|---------|--------|---------|----------|------------|
| 1 | Notes 00, 01, 03. SimuLearn replacement 01. Lab 1 | not started | | | |
| 2 | Note 02. SimuLearn replacement 02. Lab 2 | not started | | | |
| 3 | Lab 3 | not started | | | |
| 4 | Note 04. Lab 4 | not started | | | |
| 5 | Lab 5 dry run (135 min) | not started | | | |
| 6 | Cheat sheet review, exam | not started | | | |

Status values: `not started`, `in progress`, `done`, `needs redo`.

## Reading
| Note | Read | Date | Questions left |
|------|:----:|------|----------------|
| 00 mental model | [ ] | | |
| 01 AgentCore Identity | [ ] | | |
| 02 IAM refresher | [ ] | | |
| 03 JWT, OIDC, Cognito | [ ] | | |
| 04 detection | [ ] | | |
| 05 troubleshooting table | [ ] | | |
| 07 component map | [ ] | | |

## Lab steps
Tick a step only after you ran it and saw the result.

### Lab 1: Cognito + JWT authorizer
- [ ] Prerequisites checked (AWS CLI, `jq`, `uv`, `agentcore --version`)
- [ ] Cognito pool, client, user created
- [ ] Token decoded, claims table filled
- [ ] JWT runtime deployed
- [ ] Happy-path invoke returned 200
- [ ] Test A no token, B tampered, C wrong client, D wrong audience, E wrong scope recorded
- [ ] Config fixed back, 200 confirmed
- [ ] Cleanup done and verified

### Lab 2: self-hosted identity and tags
- [ ] Two workload identities created and tagged
- [ ] Token fetch tests recorded
- [ ] Tag condition tested, `aws:ResourceTag` support confirmed or refuted
- [ ] Cleanup done and verified

### Lab 3: SigV4 vs JWT
- [ ] SigV4 runtime and scoped caller policy
- [ ] Resource-based policy tested
- [ ] Scoped JWT (scope and custom claim) tested
- [ ] One authorizer type per runtime confirmed
- [ ] Cleanup done and verified

### Lab 4: detection
- [ ] Observability enabled on identity resources
- [ ] CloudTrail discovery table filled in note 04
- [ ] Four alarms built and each one fired
- [ ] Cleanup done and verified

### Lab 5: timed dry run
- [ ] Setup with seeded faults
- [ ] Tasks 1 to 7 timed (record total minutes below)
- [ ] Debrief written

## Dry run results
| Attempt | Date | Total minutes | Tasks passed (of 7) | Slowest task |
|---------|------|---------------|---------------------|--------------|
| 1 | | | | |
| 2 | | | | |

## Open questions (resolve in labs, then update the notes)
| # | Question | Where to answer | Answer | Notes updated |
|---|----------|-----------------|--------|---------------|
| 1 | Does a Cognito access token pass when `allowedAudience` is set? | Lab 1 test D | | [ ] |
| 2 | Exact status and body for each failed invoke (401 vs 403) | Lab 1, Lab 3 | | [ ] |
| 3 | Which AgentCore actions support `aws:ResourceTag`? | Lab 2, course 02 | | [ ] |
| 4 | Which Identity calls appear in CloudTrail, and as which event type? | Lab 4 | | [ ] |
| 5 | Can a runtime switch between SigV4 and JWT in place? | Lab 3 part E | | [ ] |
| 6 | Exact CLI names for tagging workload identities and fetching a workload token | Lab 2 | | [ ] |

## Session log
Newest first. One entry per session.

### Template
```
### YYYY-MM-DD HH:MM to HH:MM, Block N
- Did:
- Observed (real errors and status codes):
- Fixed in notes or labs:
- Cleanup verified: yes/no (what is left):
- Next action:
```

(No sessions yet.)
