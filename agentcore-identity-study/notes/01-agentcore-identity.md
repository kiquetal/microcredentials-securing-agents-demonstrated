# 01. AgentCore Identity

Everything below was checked against the AWS docs on 2026-11-26. Items marked "(verify in docs)" were not confirmed.

## Resources and ARNs
Source: [identity-tagging](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-tagging.html)

| Resource | ARN |
|----------|-----|
| Workload identity directory | `arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default` |
| Workload identity | `.../workload-identity-directory/default/workload-identity/NAME` |
| Token vault | `arn:aws:bedrock-agentcore:REGION:ACCOUNT:token-vault/default` |
| OAuth2 credential provider | `.../token-vault/default/oauth2credentialprovider/NAME` |
| API key credential provider | `.../token-vault/default/apikeycredentialprovider/NAME` |

Tagging support:

| Resource | Tag on create | Basic tagging | Tag-based access control (TBAC) |
|----------|:-:|:-:|:-:|
| Workload identity | yes | yes | yes |
| OAuth2 provider | yes | yes | yes |
| API key provider | yes | yes | yes |
| Directory | no | yes | yes |
| Token vault | no | yes | yes |

## Workload identity
- Agent identities are workload identities in a central directory. They are environment-agnostic.
- **Runtime creates one automatically** for each runtime. **Self-hosted agents** (EC2, ECS, EKS, on-prem) need you to create it:
  - `aws bedrock-agentcore-control create-workload-identity --name my-agent`
  - `aws bedrock-agentcore-control list-workload-identities`
  - `aws bedrock-agentcore-control delete-workload-identity --name my-agent`
  - Python: `IdentityClient(region).create_workload_identity(name=...)`
  - Source: [creating-agent-identities](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/creating-agent-identities.html)
- The `create-workload-identity` CLI reference lists a `--tags` option (verify the exact syntax with `aws bedrock-agentcore-control create-workload-identity help`).
- Service-linked role `AWSServiceRoleForBedrockAgentCoreRuntimeIdentity` (principal `runtime-identity.bedrock-agentcore.amazonaws.com`) handles workload identity permissions for agents created on or after 2025-10-13. Older agents need `GetWorkloadAccessToken*` on their execution role. The principal creating runtimes needs `iam:CreateServiceLinkedRole` for it.

## Workload access token
Source: [get-workload-access-token](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/get-workload-access-token.html)
- AWS-signed opaque token. Works only against first-party AgentCore services (for example token vault calls). Not usable against external APIs.
- Runtime and Gateway fetch it for you and pass it to your code as a payload header (`WorkloadAccessToken`).
- Three APIs and IAM actions:
  | API | Use |
  |-----|-----|
  | `GetWorkloadAccessToken` | Agent identity only, no user |
  | `GetWorkloadAccessTokenForJWT` | User proven by a JWT. AgentCore validates signature, issuer, expiry and uses `iss` + `sub`. **Recommended for production.** |
  | `GetWorkloadAccessTokenForUserId` | Caller supplies an opaque user string. Not verified by AgentCore. |
- Controls for the UserId path: restrict the IAM action to trusted principals and specific workload identities, derive the user ID from an authenticated source, log with CloudTrail, and Deny it where not needed. Partition IDs as `provider_id+user_id` (for example `cognito+user123`) when you use several IdPs.
- **Runtime-managed and Gateway-managed workload identities cannot fetch tokens directly.** The error is "WorkloadIdentity is linked to a service and cannot retrieve an access token by the caller". Self-hosted identities can.
- Python:
  ```python
  from bedrock_agentcore.services.identity import IdentityClient
  c = IdentityClient("us-east-1")
  c.get_workload_access_token(workload_name="my-agent", user_token="<jwt>")   # pattern 1
  c.get_workload_access_token(workload_name="my-agent", user_id="<id>")       # pattern 2
  ```

## Inbound auth (Runtime)
Source: [runtime-oauth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html), [inbound-jwt-authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html)

**SigV4** is the default. IAM actions: `bedrock-agentcore:InvokeAgentRuntime`, and additionally `bedrock-agentcore:InvokeAgentRuntimeForUser` when the caller sends `X-Amzn-Bedrock-AgentCore-Runtime-User-Id`. That header feeds `GetWorkloadAccessTokenForUserId`. The doc recommends denying `InvokeAgentRuntimeForUser` on runtimes that do not need it.

**JWT** (`customJWTAuthorizer`). Fields:
| Field | Checked against | Notes |
|-------|-----------------|-------|
| `discoveryUrl` | `iss` | Must end with `/.well-known/openid-configuration` (or `oauth-authorization-server`) |
| `allowedAudience` | `aud` | One token value must match one allowed value |
| `allowedClients` | `client_id` | Cognito access tokens carry `client_id`, not `aud` |
| `allowedScopes` | `scope` | At least one token scope must match |
| custom claims | named claim | `STRING` with `EQUALS`, or `STRING_ARRAY` with `CONTAINS` or `CONTAINS_ANY` |
| `allowedWorkloadConfiguration` | request identity chain | Restricts invocation to your AgentCore Gateway (by ARN or workload identity name) |

- At least one of audience, clients, scopes or custom claims is required. If several are set, **all** are verified.
- A runtime supports SigV4 **or** JWT, not both.
- For shared-issuer IdPs (GitHub Actions, GitLab) add a custom claim rule, for example on `sub`.
- Missing bearer token returns 401 with a `WWW-Authenticate` header pointing at the protected resource metadata URL.
- To read claims in agent code, add `Authorization` to the request header allowlist (`--request-header-allowlist Authorization`).

**Restrict SigV4 invocation to a gateway:** resource-based policy on the runtime with an `Allow` for the gateway role and a `Deny` with `ArnNotEquals aws:PrincipalArn` for everyone else. Also lock down who can assume that role with `aws:SourceArn` and `aws:SourceAccount` in its trust policy. See the example in the runtime-oauth page.

## Outbound auth
Source: [identity-outbound-credential-provider](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-outbound-credential-provider.html)
- **Credential providers** live in the token vault: OAuth2 (2LO client credentials, `M2M`; 3LO authorization code, `USER_FEDERATION`) and API key.
- Each provider is backed by a Secrets Manager secret in your account (see the [Secrets Manager blog](https://aws.amazon.com/blogs/machine-learning/reference-your-own-aws-secrets-manager-secrets-in-amazon-bedrock-agentcore-identity/)).
- Agent code uses decorators such as `@requires_access_token(provider_name=..., scopes=[...], auth_flow="USER_FEDERATION")` from `bedrock_agentcore.identity.auth`.
- 3LO tokens are stored with the workload identity plus the user ID (`iss` + `sub`) as the binding key, so the user is not asked for consent again until the token expires. Session binding: [oauth2-authorization-url-session-binding](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/oauth2-authorization-url-session-binding.html).
- IAM for outbound calls needs four resources: directory, workload identity, token vault, and the provider ARN. See note 02.

## Self-check answers
1. `allowedClients` answers "who may call the agent?". It checks which app (client) obtained the token.
2. Runtime-managed workload identities are blocked from fetching tokens directly. This stops agent code from extracting and misusing tokens. Runtime fetches them on the agent's behalf.
3. `bedrock-agentcore:InvokeAgentRuntimeForUser`, in addition to `bedrock-agentcore:InvokeAgentRuntime`.
