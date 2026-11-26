# Free replacement for AWS SimuLearn: Identity and Access Management

You are not using SimuLearn (paid). This replacement is aimed at what the exam needs: tag-based access, policy debugging and least privilege. You hold the security certification, so these are drills, not lessons.

| Field | Value |
|-------|-------|
| Status | not started |
| Cost | Free. Uses IAM, which has no charge |
| Fits study block | Block 2, with Lab 2 |
| Time | about 75 minutes |

## Steps
1. **Policy evaluation refresher (10 min):** [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html). Re-read the part on resource-based policies and explicit deny.
2. **ABAC hands-on (30 min):** do the IAM tutorial [Define permissions to access AWS resources based on tags](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_attribute-based-access-control.html). It gives the tag pattern used in Lab 2. Background: [What is ABAC](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html).
3. **Debug with tools (15 min):** use the [IAM policy simulator and policy testing](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html) on one of your own policies. Run [policy validation in IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-validation.html) on the Lab 2 policies once you have written them.
4. **AgentCore-specific check (10 min):** open the [Service Authorization Reference for AgentCore](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html). For each action in note 02, write down which resource types and condition keys it supports.
5. **Drills (10 min):** the three least-privilege rewrites at the bottom of [note 02](../../agentcore-identity-study/notes/02-iam-refresher.md).
6. **Then do [Lab 2](../../agentcore-identity-study/labs/lab-02-self-hosted-identity-tags.md).**

## Action support table (fill from the Service Authorization Reference)
| Action | Resource types | `aws:ResourceTag` supported? |
|--------|----------------|------------------------------|
| `GetWorkloadAccessToken` | | |
| `GetWorkloadAccessTokenForJWT` | | |
| `GetWorkloadAccessTokenForUserId` | | |
| `GetResourceOauth2Token` | | |
| `InvokeAgentRuntime` | | |
| `TagResource` | | |

## Key takeaways
-

## Errors seen and how I fixed them
| Error | Cause | Fix |
|-------|-------|-----|
| | | |

## Gaps found (move these to `../../agentcore-identity-study/cheatsheet.md`)
-
