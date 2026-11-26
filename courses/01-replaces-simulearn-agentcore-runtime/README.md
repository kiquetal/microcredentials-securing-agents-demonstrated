# Free replacement for AWS SimuLearn: Getting Started with AgentCore Runtime

You are not using SimuLearn (paid). Do this instead. It covers the same skills: deploy a runtime, see its role and workload identity, and invoke it.

| Field | Value |
|-------|-------|
| Status | not started |
| Cost | Free material. Your own AWS account pays for the resources, so clean up after each session |
| Fits study block | Block 1, before and during Lab 1 |
| Time | about 90 minutes |

## Steps
1. **Read (20 min):** [Get started with the AgentCore CLI](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-cli.html). Install the CLI and understand `agentcore create`, `build`, `deploy`, `status`.
2. **Deploy a plain runtime (30 min):** follow that page until you have a runtime with default IAM auth. Invoke it once. This is the part SimuLearn guides.
3. **Look at what was created (10 min):** the runtime's execution role and trust policy, the auto-created workload identity (`aws bedrock-agentcore-control list-workload-identities`), and the CloudWatch log group. Write down the ARNs in the table below.
4. **Add a second guided source (15 min, optional):** [onboarding repo](https://github.com/aws-samples/sample-amazon-bedrock-agentcore-onboarding). Read its runtime module and its `06_identity` README.
5. **Then do [Lab 1](../../agentcore-identity-study/labs/lab-01-cognito-jwt-runtime.md)**, which adds the JWT authorizer.
6. Clean up (see the Lab 1 cleanup checklist).

## What I found
| Item | Value (do not paste secrets) |
|------|------------------------------|
| Runtime ARN pattern | |
| Execution role name | |
| Workload identity name | |
| Auth type by default | |
| Log group | |

## Key takeaways
-

## Errors seen and how I fixed them
| Error | Cause | Fix |
|-------|-------|-----|
| | | |

## Gaps found (move these to `../../agentcore-identity-study/cheatsheet.md`)
-
