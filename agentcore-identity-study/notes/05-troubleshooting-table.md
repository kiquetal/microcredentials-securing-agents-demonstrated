# 05. Troubleshooting table

Start with this table during the dry run. Fill the "Observed" column in your own labs. Entries marked (verify) are likely causes, not confirmed behavior.

| Symptom | Likely cause | How to confirm | Fix |
|---------|--------------|----------------|-----|
| 401 on invoke, no token sent | No bearer token | Response has `WWW-Authenticate` header with `resource_metadata` | Send `Authorization: Bearer <token>` |
| 401 with token | `client_id` not in `allowedClients` | Decode token, compare | Add the client, or use the right app client |
| 401 with token | `allowedAudience` set, token has no `aud` (Cognito access token) | Decode token | Remove audience rule and use `allowedClients`, or use a token that has `aud` |
| 401 with token | Issuer mismatch | Compare `iss` with `issuer` at the discovery URL | Fix pool ID/region in `discoveryUrl` |
| 401 with token | Expired | `exp` in the past | New token |
| 401 with token | Scope or custom claim rule fails | Compare `scope` / claim values | Add scope to the app client and token, or fix the rule |
| 401 or 403 when sending a JWT to a SigV4 runtime (verify) | Runtime uses IAM auth | Check runtime authorizer type | Use the SigV4 path or deploy a JWT runtime. A runtime cannot do both |
| 403 on SigV4 invoke | Caller role lacks `bedrock-agentcore:InvokeAgentRuntime` on this runtime ARN | Policy simulator, error message | Add action on the runtime ARN only |
| 403 even with Allow | Explicit Deny, resource-based policy Deny, or tag condition | Check runtime resource policy, SCP, tag values | Fix the Deny or the tag |
| 403 with the user-ID header | Missing `bedrock-agentcore:InvokeAgentRuntimeForUser` (or an explicit Deny on it) | Policy | Grant it to trusted callers only, or stop sending the header |
| "WorkloadIdentity is linked to a service and cannot retrieve an access token by the caller" | Self-called token API on a Runtime/Gateway-managed identity | Error text | Use a self-hosted workload identity for direct calls |
| Self-hosted agent cannot get a workload access token | Identity not created, wrong name, or caller lacks `GetWorkloadAccessToken*` on directory + identity ARNs | `list-workload-identities`, policy | Create identity, fix ARNs |
| `GetResourceOauth2Token` / `GetResourceApiKey` AccessDenied | Policy lists fewer than four resources (directory, identity, vault, provider) | Policy | Add all four ARNs |
| Credentials call succeeds but no token returned (3LO) | User has not completed consent | Agent logs show authorization URL | Complete the consent flow |
| Agent can reach providers it should not | Role uses `*` for resources, or one shared role for many agents | Policy review | One identity + one role per agent, scoped ARNs, explicit Deny |
| Tag-based restriction does not work | Tag key/value mismatch, case difference, tag not on the identity, or the action does not support the condition key (verify) | `ListTagsForResource`, service authorization reference | Fix tags or condition |
| Anyone can change the tag that gates access | `TagResource`/`UntagResource` not restricted | Policy review | Restrict tagging actions and `aws:TagKeys` |
| Old agent cannot get tokens | Created before 2025-10-13, needs the token actions on its execution role | Check creation date and role | Add `GetWorkloadAccessToken*` to the execution role |
| Creating a runtime fails on identity role | Missing `iam:CreateServiceLinkedRole` for `runtime-identity.bedrock-agentcore.amazonaws.com` | Error message | Add the permission |
| No alarm fires on failures | No alarm, wrong namespace or dimension, or observability not enabled | `aws cloudwatch list-metrics --namespace AWS/Bedrock-AgentCore` | Create the alarm with correct dimensions |
| No spans or logs | Observability not enabled on the identity resource | Log group `aws/spans` empty | Enable observability on the resource |
| Event missing in CloudTrail | Data event not enabled, or not covered (verify) | Trail selectors | Enable data events or use metrics |

## 401 versus 403 rule of thumb (verify per case in labs)
- JWT runtime: the authorizer decides, so failures are 401-style authentication/authorization errors on the token.
- SigV4 runtime: IAM decides, so failures are 403 AccessDenied-style.
Record the real status codes and bodies in Lab 1 and Lab 3.
