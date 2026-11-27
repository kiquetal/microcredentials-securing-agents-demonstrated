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

## Runtime-hosted vs self-hosted agents
Runtime runs your agent code. Identity names the agent and handles tokens and credentials. Runtime uses Identity, but Identity also works for agents that do not run on Runtime.

```
CASE A: hosted agent (Runtime does the work for you)

 caller --JWT--> +-------------------------+
                 | AGENTCORE RUNTIME       |   (compute: microVM per session)
                 |  1. checks the JWT      |
                 |  2. fetches token  -----+------>  +-------------------------+
                 |  3. runs your code      |         | AGENTCORE IDENTITY      |
                 |     (gets token in a    | <------ |  workload identity      |
                 |      header)            |  token  |  (AUTO-CREATED)         |
                 +-------------------------+         |  token vault            |
                                                     +-------------------------+
 You configure: authorizer only. Runtime creates the identity and fetches
 the tokens. Your code cannot fetch them itself.


CASE B: self-hosted agent (you do the work)

 +--------------------------+                        +-------------------------+
 | EC2 / ECS / EKS /        |  your IAM role calls   | AGENTCORE IDENTITY      |
 | on-prem / laptop         | ---------------------> |  workload identity      |
 | your agent code          |  GetWorkloadAccessToken|  (YOU CREATE IT:        |
 | (no AgentCore Runtime)   | <--------------------- |   create-workload-      |
 +--------------------------+         token          |   identity)             |
                                                     +-------------------------+
 You configure: the identity, the tags, and the IAM role that may fetch tokens.
```

| | Runtime-hosted | Self-hosted |
|---|---|---|
| Where the code runs | AgentCore Runtime | Your own EC2, ECS, EKS, on-prem or laptop |
| Workload identity | Created automatically | You create it |
| Fetching workload access tokens | Runtime does it. Agent code is blocked from calling the API. | Your code or role calls the API |
| Who decides access | Authorizer plus IAM | Your IAM role, scoped to the identity ARN, plus tags |

Exam objective 1 says "provision identities for self-hosted agents" because nobody creates the identity for you in Case B. Lab 1 covers Case A. Lab 2 covers Case B.

**Provision once, fetch per request.** A workload identity is a long-lived registration. You create it once per agent (one identity and one IAM role per agent). Do not create one per request or per user. At run time the agent fetches a workload access token from that identity. For a user request it calls `GetWorkloadAccessTokenForJWT`, which adds the user (`iss` + `sub`) to the token. The identity stays the same. Token lifetime is not stated in the notes (verify in docs).

### Sequence: self-hosted agent with tag-based access (Lab 2)
```
   Admin                IAM           AgentCore Identity       Agent (role A)
     |                   |                     |                      |
   SETUP: once per agent, by an admin          |                      |
     | 1. create-workload-identity lab-agent-a, tag Team=red          |
     |---------------------------------------->|                      |
     | 2. create role LabAgentRoleA, tag Team=red                     |
     |------------------>|                     |                      |
     | 3. Allow GetWorkloadAccessToken on directory + identity/*      |
     |------------------>|                     |                      |
     | 4. if ResourceTag/Team = PrincipalTag/Team; Deny TagResource   |
     |------------------>|                     |                      |
   RUN TIME: every request                     |                      |
     |                   |                     | 5. GetWorkloadAccessToken(lab-agent-a)
     |                   |                     |<---------------------|
     |                   | 6. check: identity tag Team=red, role tag Team=red
     |                   |<--------------------|                      |
     |                   | 7. tags match, allow|                      |
     |                   |-------------------->|                      |
     |                   |                     | 8. workload access token
     |                   |                     |--------------------->|
     |                   |                     | 9. GetWorkloadAccessToken(lab-agent-b, tagged Team=blue)
     |                   |                     |<---------------------|
     |                   |                     | 10. AccessDenied: tags differ
     |                   |                     |--------------------->|
     |                   |                     |                      |
```
Steps 3 and 4 are the pattern from note 02. Whether each AgentCore action supports `aws:ResourceTag` is not confirmed (verify in docs and in Lab 2). Step 6 is a simplification: IAM evaluates the caller's policy when the API call arrives.

### Sequence: Runtime-hosted agent (Lab 1)
```
  Client              Runtime         AgentCore Identity         Agent code
     |                   |                     |                      |
   SETUP: once, by the developer               |                      |
     | 1. agentcore add runtime ... --authorizer-type CUSTOM_JWT      |
     |------------------>|                     |                      |
     |                   | 2. Runtime creates the workload identity for you
     |                   |-------------------->|                      |
   RUN TIME: every request                     |                      |
     | 3. InvokeAgentRuntime, Authorization: Bearer <JWT>             |
     |------------------>|                     |                      |
   4. authorizer checks iss, client_id/aud, scope, claims (401 if any fails)
     |                   | 5. GetWorkloadAccessTokenForJWT (iss + sub)|
     |                   |-------------------->|                      |
     |                   | 6. workload access token                   |
     |                   |<--------------------|                      |
     |                   | 7. run your code, token in header WorkloadAccessToken
     |                   |------------------------------------------->|
     |                   |                     | 8. token vault call, e.g. GetResourceOauth2Token (later labs)
     |                   |                     |<---------------------|
     |                   |                     | 9. agent code calls GetWorkloadAccessToken itself
     |                   |                     |<---------------------|
     |                   |                     | 10. error: identity is linked to a service
     |                   |                     |--------------------->|
     |                   |                     |                      |
```
In step 10 the full error text is "WorkloadIdentity is linked to a service and cannot retrieve an access token by the caller".

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
