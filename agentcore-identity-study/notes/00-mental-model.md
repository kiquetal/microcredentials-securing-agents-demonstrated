# 00. Mental model

Four questions cover the whole exam. Every task in the lab maps to one of them.

| Question | Mechanism | Where you configure it |
|----------|-----------|------------------------|
| Who is the agent? | Workload identity in the AgentCore Identity directory | Auto-created for Runtime. `create-workload-identity` for self-hosted agents |
| Who may call the agent? (inbound) | IAM SigV4 **or** JWT bearer authorizer | Runtime `authorizerConfiguration`, plus IAM / resource-based policy |
| What may the agent call? (outbound) | IAM role, OAuth2 credential providers (2LO / 3LO), API key providers, token vault | Credential providers + IAM policy on the agent role |
| Who did what? | CloudTrail, CloudWatch metrics, spans and logs | Trail, metric filters, alarms |

## Flow (hosted agent, JWT inbound)
```
User -> IdP (Cognito) -> access token (JWT)
     -> InvokeAgentRuntime with Authorization: Bearer <JWT>
Runtime authorizer: checks issuer, signature, expiry, plus aud / client_id / scope / custom claims
Runtime -> GetWorkloadAccessTokenForJWT (identity = iss + sub of the user, plus the agent's workload identity)
Runtime -> passes a workload access token to the agent code
Agent -> token vault (GetResourceOauth2Token / GetResourceApiKey) -> third-party API
```
Source: [runtime-oauth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html), [get-workload-access-token](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/get-workload-access-token.html).

## AuthN vs AuthZ here
- **Authentication** at the Runtime door: signature and issuer of the JWT, or the SigV4 signature.
- **Authorization** at the Runtime door: IAM policy (SigV4 path), or the allowed audience, client, scopes and custom claims (JWT path).
- Authorization for outbound calls is IAM on the agent's role plus the user consent stored in the token vault.

## SigV4 vs JWT in one line
- SigV4: caller is an AWS principal. Access is decided by IAM identity and resource policies. Works with no extra setup.
- JWT: caller is a user or app from an OIDC/OAuth IdP. Access is decided by the authorizer config. A runtime supports **one** of the two at a time, not both. Use separate runtime versions or runtimes if you need both.

## Self-check (answer from memory)
1. Which of the four questions does `allowedClients` answer?
2. Why can a runtime-hosted agent not call `GetWorkloadAccessToken` itself?
3. Name the exact IAM action needed when the `X-Amzn-Bedrock-AgentCore-Runtime-User-Id` header is used.

Answers are in [01-agentcore-identity.md](01-agentcore-identity.md).
