# Study plan: AWS Securing Agent Identities Demonstrated

Start date: Thu 2026-11-26. Pace: about 2 weeks, one block of 1.5 to 2 hours per session. No fixed exam date.

## Assumptions about you
- You hold an AWS Security certification, so IAM, STS, CloudTrail and CloudWatch basics are skipped. `notes/02-iam-refresher.md` only lists the AgentCore-specific parts.
- You have your own AWS account. Paid SimuLearn and Cloud Quest items are optional and not part of this plan.
- Weakest area: JWT/OIDC and how AgentCore validates tokens. Spend extra time on `notes/03-jwt-oidc-cognito.md` and Lab 1.

## The exam in short
Sources: see `notes/06-references.md`. Confirm on the Skill Builder page before booking.
- Hands-on lab, 135 minutes, AWS-provisioned environment, no hints, no multiple choice. Free.
- Objectives:
  1. Restrict workload identity access with tags. Provision identities for self-hosted agents.
  2. Configure scoped JWT authorization and a SigV4 authentication path.
  3. Detect identity security events with CloudTrail and CloudWatch.
- Credly badge, valid 12 months. Wait 25 days after a failed attempt.

## Method: read, then break and fix
The exam is a fix-it lab, so reading alone will not pass it. For each topic:
1. Read the note (10 to 20 minutes).
2. Do the matching lab. Build it once the documented way.
3. Break it on purpose (wrong audience, missing tag, wrong principal), read the exact error, and fix it.
4. Write the symptom and fix in `cheatsheet.md`.

## Reading order
| # | File | Why |
|---|------|-----|
| 1 | [notes/00-mental-model.md](notes/00-mental-model.md) | One-page map of who/what/how |
| 2 | [notes/01-agentcore-identity.md](notes/01-agentcore-identity.md) | Core service concepts |
| 3 | [notes/03-jwt-oidc-cognito.md](notes/03-jwt-oidc-cognito.md) | Your weak spot, so read early |
| 4 | [notes/02-iam-refresher.md](notes/02-iam-refresher.md) | AgentCore-specific IAM only |
| 5 | [notes/04-detection-cloudtrail-cloudwatch.md](notes/04-detection-cloudtrail-cloudwatch.md) | Objective 3 |
| 6 | [notes/05-troubleshooting-table.md](notes/05-troubleshooting-table.md) | Symptom to fix |
| 7 | [notes/06-references.md](notes/06-references.md) | Links and video search terms |

## Schedule

### Block 1: today, 2 hours (about 20:30 to 22:30)
| Time | Activity |
|------|----------|
| 0:00 - 0:10 | Confirm CLI profile and region (`aws sts get-caller-identity`). Install or update the AWS CLI, `jq`, and the AgentCore CLI (see Lab 1 prerequisites). Optional: a budget alert if you do not already have one. |
| 0:10 - 0:40 | Read notes 00, 01 and 03. Open the official [runtime-oauth page](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html) beside them. |
| 0:40 - 1:45 | [Lab 1](labs/lab-01-cognito-jwt-runtime.md): Cognito, JWT authorizer, deploy, invoke with a valid token, a wrong client, and a bad token. |
| 1:45 - 2:00 | Fill the Lab 1 results table. Add three lines to `cheatsheet.md`. Run the cleanup checklist. |

### Later blocks
| Block | Content | Lab |
|-------|---------|-----|
| 2 | Self-hosted workload identity, workload access token, tag-based IAM (objective 1). Read note 02. | [Lab 2](labs/lab-02-self-hosted-identity-tags.md) |
| 3 | SigV4 path versus JWT path, resource-based policy, one authorizer type per runtime (objective 2). | [Lab 3](labs/lab-03-sigv4-vs-jwt.md) |
| 4 | Detection with CloudTrail and CloudWatch (objective 3). Read note 04. | [Lab 4](labs/lab-04-detection.md) |
| 5 | Timed 135-minute dry run. Ask a friend or write the scenario yourself, then break and fix. | [Lab 5](labs/lab-05-timed-dry-run.md) |
| 6 | Review `cheatsheet.md`, redo your slowest lab step, then take the exam. | none |

If you have more time, add a second dry run before Block 6. Speed matters: 135 minutes is tight for several fixes.

## Rules
- Delete resources at the end of every block. Each lab ends with a cleanup checklist.
- Do all work in one test region (suggested: `us-east-1`). Check AgentCore availability in your region.
- Mark anything you could not confirm with "(verify in docs)" in your own notes too.
- Do not paste real tokens or secrets into these files.

## Progress checklist
- [ ] Block 1: notes 00, 01, 03 read. Lab 1 done and cleaned up.
- [ ] Block 2: note 02 read. Lab 2 done and cleaned up.
- [ ] Block 3: Lab 3 done and cleaned up.
- [ ] Block 4: note 04 read. Lab 4 done and cleaned up.
- [ ] Block 5: Lab 5 dry run finished within 135 minutes.
- [ ] `cheatsheet.md` filled in.
- [ ] Exam taken.

## Files
- [cheatsheet.md](cheatsheet.md)
- Notes: [00](notes/00-mental-model.md), [01](notes/01-agentcore-identity.md), [02](notes/02-iam-refresher.md), [03](notes/03-jwt-oidc-cognito.md), [04](notes/04-detection-cloudtrail-cloudwatch.md), [05](notes/05-troubleshooting-table.md), [06](notes/06-references.md)
- Labs: [01](labs/lab-01-cognito-jwt-runtime.md), [02](labs/lab-02-self-hosted-identity-tags.md), [03](labs/lab-03-sigv4-vs-jwt.md), [04](labs/lab-04-detection.md), [05](labs/lab-05-timed-dry-run.md)
