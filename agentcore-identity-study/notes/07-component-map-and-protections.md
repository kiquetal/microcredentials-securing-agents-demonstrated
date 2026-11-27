# 07. Component map and how to protect each connection

Use this note to answer two exam-style questions: which AgentCore component talks to which, and what protects that connection. Sources are listed at the end. Anything not confirmed in the docs is marked "(verify in docs)".

## 1. The components

```
+------------------------------------------------------------------------------+
|                           AMAZON BEDROCK AGENTCORE                           |
|                                                                              |
|  BUILD / RUN              CONNECT                    SECURE / OBSERVE        |
|  +-----------------+      +--------------------+     +--------------------+  |
|  | RUNTIME         |      | GATEWAY            |     | IDENTITY           |  |
|  | hosts agent     |      | tools as MCP,      |     | workload identity  |  |
|  | code in a       |      | single front door  |     | directory, token   |  |
|  | microVM/session |      | (inbound + egress) |     | vault, credential  |  |
|  +-----------------+      +--------------------+     | providers          |  |
|  +-----------------+      +--------------------+     +--------------------+  |
|  | MEMORY          |      | POLICY             |     +--------------------+  |
|  | short and long  |      | policy engine for  |     | OBSERVABILITY      |  |
|  | term agent data |      | gateway targets    |     | metrics, spans,    |  |
|  +-----------------+      | (verify in docs)   |     | logs (CloudWatch)  |  |
|  +-----------------+      +--------------------+     +--------------------+  |
|  | CODE INTERPRETER|      +--------------------+     +--------------------+  |
|  | sandboxed code  |      | REGISTRY           |     | EVALUATIONS        |  |
|  +-----------------+      | agent / tool       |     | quality scoring    |  |
|  +-----------------+      | catalog            |     | (verify in docs)   |  |
|  | BROWSER         |      | (verify in docs)   |     +--------------------+  |
|  | managed browser |      +--------------------+                             |
|  +-----------------+                                                         |
+------------------------------------------------------------------------------+
        ^ Identity is the one that every other component relies on for
          "who is this agent" and "which credentials may it get".
```
The list of services comes from the AWS sample course that uses "all 9 AgentCore services" and from the docs pages. The exam focus is **Identity, Runtime and Gateway**, plus **Observability** for detection.

## 2. Who talks to whom (data path and trust boundaries)

```
                      TRUST BOUNDARY A: caller -> AgentCore
 +--------+  (1) JWT bearer  or  SigV4
 | User / |------------------------------+
 | client |                              v
 +--------+                    +-------------------+     (2) JWT or SigV4
     ^                         |  AGENTCORE        |--------------------+
     |                         |  GATEWAY          |                    |
     | IdP token (Cognito...)  |  inbound authz    |                    v
 +----+-----+                  |  policy, guard-   |          +-------------------+
 |   IdP    |                  |  rails, intercept.|          |  AGENTCORE        |
 | (OIDC)   |                  +-------------------+          |  RUNTIME          |
 +----------+                      | (5) outbound auth        |  microVM / session|
                                   v                          |  execution role   |
                   +------------------------------+           +---------+---------+
                   | TARGETS: Lambda, OpenAPI,     |                    |
                   | Smithy, another runtime, MCP  |   (3) workload     |
                   +------------------------------+       access token  |
                                                                       v
                         TRUST BOUNDARY B                  +-------------------+
                         agent -> credentials              |  AGENTCORE        |
                                                           |  IDENTITY         |
  +----------------------+   (4) provider ARN + token      |  directory        |
  | Third-party API      |<--------------------------------|  token vault      |
  | (Google, Slack...)   |        OAuth2 2LO/3LO, API key  |  (Secrets Manager)|
  +----------------------+                                 +-------------------+

  Runtime also calls AWS services (Bedrock models, S3...) with its EXECUTION ROLE.
  Everything emits metrics/spans/logs to CloudWatch, and API calls go to CloudTrail.
```

| # | Connection | Authentication | Authorization |
|---|------------|----------------|---------------|
| 1 | Client to Gateway | OAuth JWT, IAM SigV4, "authenticate only", or none (dev only) | Authorizer config, IAM policy, gateway policy engine |
| 2 | Client or Gateway to Runtime | SigV4 **or** JWT (one per runtime, not both) | IAM + resource policy, or audience/client/scope/claims |
| 3 | Runtime to Identity | Workload access token (AWS-signed, issued for the runtime's workload identity) | IAM on token actions, workload identity ARN |
| 4 | Agent to third-party API | OAuth2 token or API key held in the token vault | IAM on `GetResourceOauth2Token` / `GetResourceApiKey` (four ARNs), user consent for 3LO |
| 5 | Gateway to target | Gateway service role (`GATEWAY_IAM_ROLE`), OAuth, or API key via credential providers | Role policy, provider scoping |
| 6 | Runtime to AWS services | Execution role credentials (SigV4) | Role policy |

## 2b. Runtime-hosted vs self-hosted agents
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

## 3. How to protect each connection

```
 [1] Client -> Gateway / Runtime
     - Use JWT with ALL fields: discovery URL, audience, client, scope, custom claims
     - Prefer JWT over the user-ID header for user identity
     - Deny InvokeAgentRuntimeForUser where not needed
     - Never leave "no authorization" on a gateway outside dev

 [2] Gateway -> Runtime  (stop callers from bypassing the gateway)
     - SigV4 runtime : resource policy: Allow gateway role, Deny everyone else
                       (ArnNotEquals aws:PrincipalArn)
     - JWT runtime   : allowedWorkloadConfiguration with the gateway ARN
     - Trust policy of the gateway role: aws:SourceArn + aws:SourceAccount

 [3] Runtime <-> Identity
     - Runtime-managed workload identities cannot fetch tokens directly
     - Self-hosted: one workload identity + one IAM role per agent
     - Tag the identities and use tag conditions; restrict TagResource
     - Deny GetWorkloadAccessTokenForUserId where a JWT is always available

 [4] Agent -> third-party API
     - Four-resource IAM policy per provider (directory, identity, vault, provider)
     - Explicit Deny on sensitive providers
     - Keep keys in the token vault, not in code, env vars or logs
     - 3LO for user-delegated access, 2LO for autonomous access

 [5] Gateway -> targets
     - Least-privilege gateway role per target; scoped credential providers
     - Interceptors and policy engine for allow/deny per tool (verify in docs)

 [6] Runtime -> AWS services
     - Least-privilege execution role (the CLI default allows all foundation models)
     - Anything in the microVM can reach the role credentials via MMDS
     - Execution role must not exceed the privileges of those who can invoke it
     - Trust policy: aws:SourceAccount and aws:SourceArn (confused deputy)
```

## 4. Network and data protection

```
  Client / VPC
      |  TLS 1.2+ (TLS 1.3 recommended)
      v
  [ PrivateLink endpoints ]  com.amazonaws.REGION.bedrock-agentcore          (data plane)
                             com.amazonaws.REGION.bedrock-agentcore-control  (control plane)
      |  endpoint policy: SigV4 -> list IAM principals; OAuth -> Principal "*"
      v
  Runtime in your VPC (private subnets, NAT for internet), security groups least privilege
      |
      +--> private DBs / internal APIs (VPC connectivity)
      +--> VPC condition keys: bedrock-agentcore:subnets, bedrock-agentcore:securityGroups
```
- Isolation: each session runs in its own microVM. Memory is sanitized after the session ends. The VM is the isolation boundary.
- Session binding: with platform V1 or Instances, your backend must tie each user to their session IDs. V2 on microVMs pins sessions created with a user identity.
- Encryption: TLS in transit, AWS-owned KMS keys at rest by default. The token vault can use a customer-managed key (verify in docs).
- Private IdP: Identity can reach an OIDC IdP inside your VPC without public exposure ([private IdP page](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-private-idp.html)).

## 5. Application-layer protections that people forget
- Validate that `prompt` is a string. A non-string value with a `toolUse` block can make the agent framework run a tool without model reasoning.
- Strip `toolUse` blocks from user-supplied message history.
- Run containers as non-root. Restrict agent tools from calling `localhost` (platform server).
- Restrict `InvokeAgentRuntimeCommand` and `InvokeAgentRuntimeCommandShell` to a small set of principals.
- MMDSv2 must be enabled on runtimes (the docs state a June 30, 2026 deadline).

## 6. Detection per connection
| Connection | Where to look |
|------------|---------------|
| Client to Runtime | CloudTrail `InvokeAgentRuntime` (check `runtimeUserId`), runtime log group |
| Token fetches | `AWS/Bedrock-AgentCore` metrics `WorkloadAccessTokenFetch*`, spans in `aws/spans` |
| Provider calls | `ResourceAccessTokenFetchFailures`, `ApiKeyFetchFailures` |
| Gateway | CloudTrail management and data events for Gateway (documented) |
| Network | VPC Flow Logs for VPC-connected runtimes |

## 7. Self-check
1. In the diagram, which two connections need a resource policy or `allowedWorkloadConfiguration` to stop bypass?
2. Which connection uses the four-resource IAM policy?
3. Why must the execution role be equal to or smaller than the privileges of its callers?
4. Which VPC endpoint policy `Principal` do you use for OAuth callers?

Answers: 1) client-to-runtime when a gateway should be the only caller (connection 2 in the table); 2) agent to third-party API via the token vault (connection 4); 3) otherwise a caller can use the agent to escalate; 4) `*`, because endpoint policies cannot match OAuth users.

## Sources
- [Security best practices for AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-security-best-practices.html)
- [Inbound and outbound auth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html)
- [Gateway core concepts](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-core-concepts.html) (inbound types: OAuth JWT, IAM SigV4, authenticate only, none)
- [Gateway overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html) (OpenAPI, Smithy, Lambda targets)
- [Identity observability data](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-identity-metrics.html)
- [Scope down access to credential providers](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/scope-credential-provider-access.html)
- [Complete AgentCore course (9 services)](https://github.com/mikegc-aws/amazon-bedrock-agentcore-complete-course)
