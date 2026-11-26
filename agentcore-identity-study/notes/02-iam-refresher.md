# 02. IAM refresher (AgentCore-specific only)

You already know policy evaluation, trust policies, ABAC and condition operators from the Security certification. This note only covers the AgentCore-specific parts you must apply in the lab.

## The four-resource rule for outbound credentials
Source: [scope-credential-provider-access](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/scope-credential-provider-access.html)

`GetResourceOauth2Token` and `GetResourceApiKey` need all four resources in the `Resource` list. Leaving one out causes AccessDenied.

```json
{
  "Effect": "Allow",
  "Action": "bedrock-agentcore:GetResourceOauth2Token",
  "Resource": [
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/WORKLOAD_NAME",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:token-vault/default",
    "arn:aws:bedrock-agentcore:REGION:ACCOUNT:token-vault/default/oauth2credentialprovider/PROVIDER_NAME"
  ]
}
```
For API keys use `GetResourceApiKey` and `apikeycredentialprovider/PROVIDER_NAME`.

Facts from the doc:
- The service does **not** bind a workload identity to a credential provider by itself. Only the IAM role of the agent decides which providers it can call.
- Use one workload identity and one IAM role per agent when agents need different providers.
- An explicit `Deny` on the provider ARNs is the recommended guardrail for sensitive providers.
- A successful call does not guarantee credentials. For 3LO the user must have consented first.

## Workload access token actions
`bedrock-agentcore:GetWorkloadAccessToken`, `...ForJWT`, `...ForUserId`. Scope to the directory and the workload identity ARNs:
```
arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default
arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/NAME
```
To block the user-ID path where a JWT is always available:
```json
{ "Effect": "Deny", "Action": "bedrock-agentcore:GetWorkloadAccessTokenForUserId", "Resource": "*" }
```

## Invocation actions
- `bedrock-agentcore:InvokeAgentRuntime` on `arn:aws:bedrock-agentcore:REGION:ACCOUNT:runtime/RUNTIME_ID`
- `bedrock-agentcore:InvokeAgentRuntimeForUser` for callers that pass the user-ID header. Deny it by default:
  ```json
  { "Effect": "Deny", "Action": "bedrock-agentcore:InvokeAgentRuntimeForUser",
    "Resource": "arn:aws:bedrock-agentcore:REGION:ACCOUNT:runtime/*" }
  ```

## Tag-based access control (objective 1)
- Workload identities, OAuth2 providers and API key providers support tags at creation. The directory and the token vault can be tagged afterwards. All five support TBAC.
- Typical pattern: tag the workload identity (`Team=payments`) and write an IAM policy that allows the action only when the tag matches:
  ```json
  {
    "Effect": "Allow",
    "Action": "bedrock-agentcore:GetWorkloadAccessToken",
    "Resource": "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/*",
    "Condition": { "StringEquals": { "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}" } }
  }
  ```
  This is a pattern, not a verified AWS example. Whether each AgentCore action supports `aws:ResourceTag` is listed in the Service Authorization Reference. Check it for your actions (verify in docs): <https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html>
- Keep tagging itself under control: restrict `bedrock-agentcore:TagResource`, `UntagResource` (and `aws:TagKeys` / `aws:RequestTag`) so a caller cannot change the tag that grants access.
- Tag-based access control questions to ask in the lab: which principal tag, which resource tag, and who can change either one?

## Confused-deputy conditions
Trust policies for roles assumed by AgentCore services should carry `aws:SourceAccount` and `aws:SourceArn`. See the "Confused deputy prevention" section in [runtime-security-best-practices](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-security-best-practices.html).

## Resource-based policy on a runtime
Used to restrict SigV4 invocation. Pattern: `Allow` the intended role, plus `Deny` with `ArnNotEquals` on `aws:PrincipalArn` for everyone else. An explicit deny overrides identity-based allows in the same account. Docs: [resource-based-policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html).

## STS GetScopedCredentials
New STS API that scopes down existing temporary credentials. Possible but unconfirmed exam topic. Read the [AWS blog post](https://aws.amazon.com/blogs/security/scope-down-permissions-of-existing-sessions-using-aws-sts-getscopedcredentials/) only after the labs.

## Least-privilege rewrite drills
Rewrite each statement so it is safe:
1. `bedrock-agentcore:*` on `*` for an agent role. Reduce to the token-fetch and provider actions it uses, on its own workload identity and provider ARNs.
2. `InvokeAgentRuntime` on `runtime/*` for a CI role. Reduce to one runtime ID.
3. A trust policy for `bedrock-agentcore.amazonaws.com` with no conditions. Add `aws:SourceAccount` and `aws:SourceArn`.
