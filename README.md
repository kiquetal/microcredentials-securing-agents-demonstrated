# AWS Securing Agent Identities Demonstrated: study kit

Notes, hands-on labs, a study plan and a cheat sheet for the free AWS Skill Builder microcredential **AWS Securing Agent Identities Demonstrated**. The kit uses only free material: AWS docs, AWS sample repos and your own AWS account. It does not need paid SimuLearn courses.

## About the exam
Details come from third-party summaries and AWS announcements. Confirm them on the AWS Skill Builder assessment page before booking.

- Hands-on lab, 135 minutes, in an AWS-provisioned environment. No hints, no multiple choice. Free.
- Objectives:
  1. Restrict workload identity access with tags, and provision identities for self-hosted agents.
  2. Configure scoped JWT authorization and a SigV4 authentication path.
  3. Detect identity security events with CloudTrail and CloudWatch.
- Credly badge, valid 12 months. A failed attempt means a 25-day wait.

## Where to start
Open [agentcore-identity-study/plans.md](agentcore-identity-study/plans.md). It has the reading order, a 2-hour first block, the later blocks and a progress checklist.

The method is read, build, break on purpose, then fix. The exam is a fix-it lab, so reading alone will not be enough.

## Progress
This README is the main status file. Update it whenever a block, lab or note changes. Last updated: 2026-11-26.

| Block | Content | Status |
|-------|---------|--------|
| 1 | Notes 00, 01, 03 and Lab 1 | not started |
| 2 | Note 02 and Lab 2 (self-hosted identity, tags) | not started |
| 3 | Lab 3 (SigV4 vs JWT) | not started |
| 4 | Note 04 and Lab 4 (detection) | not started |
| 5 | Lab 5 (timed dry run) | not started |
| 6 | Cheat sheet review and exam | not started |

## Contents
| Path | What it is |
|------|------------|
| [plans.md](agentcore-identity-study/plans.md) | Study plan, schedule, checklist |
| [cheatsheet.md](agentcore-identity-study/cheatsheet.md) | One-page sheet you fill in as you study |
| [notes/00-mental-model.md](agentcore-identity-study/notes/00-mental-model.md) | The four questions: who is the agent, who may call it, what may it call, who did what |
| [notes/01-agentcore-identity.md](agentcore-identity-study/notes/01-agentcore-identity.md) | Workload identities, workload access tokens, inbound and outbound auth |
| [notes/02-iam-refresher.md](agentcore-identity-study/notes/02-iam-refresher.md) | AgentCore-specific IAM, tag-based access, resource policies |
| [notes/03-jwt-oidc-cognito.md](agentcore-identity-study/notes/03-jwt-oidc-cognito.md) | JWT claims, OIDC discovery, Cognito, common 401 causes |
| [notes/04-detection-cloudtrail-cloudwatch.md](agentcore-identity-study/notes/04-detection-cloudtrail-cloudwatch.md) | Metrics, spans, CloudTrail, alarm patterns |
| [notes/05-troubleshooting-table.md](agentcore-identity-study/notes/05-troubleshooting-table.md) | Symptom, cause, fix |
| [notes/06-references.md](agentcore-identity-study/notes/06-references.md) | All links and video search terms |
| [labs/lab-01-cognito-jwt-runtime.md](agentcore-identity-study/labs/lab-01-cognito-jwt-runtime.md) | Cognito, JWT authorizer, deploy, break and record errors |
| [labs/lab-02-self-hosted-identity-tags.md](agentcore-identity-study/labs/lab-02-self-hosted-identity-tags.md) | Self-hosted workload identity and tag-based access |
| [labs/lab-03-sigv4-vs-jwt.md](agentcore-identity-study/labs/lab-03-sigv4-vs-jwt.md) | SigV4 versus JWT, resource policy, scoped JWT |
| [labs/lab-04-detection.md](agentcore-identity-study/labs/lab-04-detection.md) | CloudTrail and CloudWatch detection |
| [labs/lab-05-timed-dry-run.md](agentcore-identity-study/labs/lab-05-timed-dry-run.md) | 135-minute timed dry run |

## Requirements
- An AWS account where you can create Cognito pools, IAM roles, AgentCore runtimes and workload identities. A Bedrock model enabled for the sample agent in Lab 1.
- AWS CLI (latest), `jq`, `python3`, and the [AgentCore CLI](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-cli.html).
- Labs create billable resources. Each lab ends with a cleanup checklist. Run it at the end of every session.

## Accuracy
Facts were checked against the AWS developer guide on 2026-11-26. Anything not confirmed is marked "(verify in docs)" in the notes. Known open points:
- Which AgentCore Identity and Runtime calls appear in CloudTrail. The docs I read list CloudTrail coverage only for Gateway and Agent Registry. Lab 4 has you find out in your own account.
- Which AgentCore IAM actions support `aws:ResourceTag` conditions. Check the [Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html).
- Some lab commands are marked "check with `help`" because I could not confirm the exact CLI names. The service and CLI change often, so trust the current AWS docs over these notes.

## Security
Do not commit tokens, passwords or account IDs. The labs tell you not to print tokens.

## Status
Version `v0.1.0`: first complete draft. The notes and labs have not yet been run end to end against a live account.

This is an unofficial study aid and is not affiliated with AWS.
