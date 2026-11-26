# 04. Detection with CloudTrail and CloudWatch (objective 3)

Objective: "Detect identity security events with AWS CloudTrail and Amazon CloudWatch." You already know CloudTrail and CloudWatch. The open question is which AgentCore Identity data lands where. Verify each item in your own account during Lab 4.

## What the docs confirm
**CloudWatch metrics.** Source: [observability-identity-metrics](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-identity-metrics.html)

Namespace `AWS/Bedrock-AgentCore`:
| Metric | Dimensions |
|--------|------------|
| `WorkloadAccessTokenFetchSuccess` | WorkloadIdentity, WorkloadIdentityDirectory, Operation |
| `WorkloadAccessTokenFetchFailures` | same + ExceptionType |
| `WorkloadAccessTokenFetchThrottles` | WorkloadIdentity, WorkloadIdentityDirectory, Operation |
| `ResourceAccessTokenFetchSuccess` / `Failures` / `Throttles` | WorkloadIdentity, WorkloadIdentityDirectory, TokenVault, ProviderName, Type (+ ExceptionType on Failures) |
| `ApiKeyFetchSuccess` / `Failures` / `Throttles` | WorkloadIdentity, WorkloadIdentityDirectory, TokenVault, ProviderName (+ ExceptionType on Failures) |

Namespace `AWS/Usage`: `CallCount`, `ThrottleCount` (dimensions Service, Type, Class, Resource).

**Spans and logs.** You must enable observability on the workload identity or credential provider resource.
- Spans go to the `aws/spans` log group. Operations: `GetWorkloadAccessToken`, `GetWorkloadAccessTokenForJWT` (adds `issuer`, `user_sub`), `GetWorkloadAccessTokenForUserId`, `GetResourceOAuth2Token`, `GetResourceAPIKey`. Attributes include `workload.identity.id`, `error_type`, `latency_ms`.
- Application logs have fields `resource_arn`, `account_id`, `request_id`, `trace_id`, `span_id`, `service_name` (`BedrockAgentCore.Identity`), `operation`, `request_payload`, `response_payload`. They can go to CloudWatch Logs, S3 or Firehose.

**CloudTrail.** The [observability overview](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability.html) says CloudTrail coverage is currently documented for **AgentCore Gateway** and **Agent Registry** only.
- The runtime-oauth page and the workload-token page both tell you to use CloudTrail to watch `InvokeAgentRuntime` calls that include `runtimeUserId`, and `GetWorkloadAccessTokenForUserId` calls.
- Whether Identity and Runtime data-plane calls appear as management events, data events or not at all is **not confirmed in the docs I read (verify in docs and in your account)**.

## Lab 4 discovery tasks (do these, record the answers here)
| Question | Answer from your account |
|----------|--------------------------|
| Does `CreateWorkloadIdentity` appear in CloudTrail event history? Event source name? | |
| Does `DeleteWorkloadIdentity`? | |
| Does `TagResource` on a workload identity? | |
| Does `InvokeAgentRuntime` appear? Management or data event? Does it include `runtimeUserId`? | |
| Do `GetWorkloadAccessToken*` calls appear? | |
| Is a data event resource type available in the trail's advanced event selectors? Which? | |
| Which identity events does a denied call produce (`errorCode`)? | |

Find event names with:
```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock-agentcore.amazonaws.com --max-results 20
```
Event history only holds management events from the last 90 days. If an event is missing there, check whether it is a data event and enable it on a trail (see the Gateway example in [enabling-cloudtrail-data-event-logging](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/enabling-cloudtrail-data-event-logging.html) for the shape of the configuration).

## Alarm patterns to build
1. **Token fetch failures:** CloudWatch alarm on `WorkloadAccessTokenFetchFailures` (namespace `AWS/Bedrock-AgentCore`), statistic Sum, threshold 1 in 5 minutes. Add the `ExceptionType` dimension to separate access denied from throttling.
2. **Throttles:** alarm on `WorkloadAccessTokenFetchThrottles`.
3. **Provider failures:** alarm on `ResourceAccessTokenFetchFailures` or `ApiKeyFetchFailures` per provider.
4. **Risky actions via CloudTrail:** metric filter or EventBridge rule on `GetWorkloadAccessTokenForUserId`, `InvokeAgentRuntimeForUser`, `CreateWorkloadIdentity`, `DeleteWorkloadIdentity`, tag changes (confirm event names first). Requires CloudTrail delivery to CloudWatch Logs.
5. **Spans:** Logs Insights query on `aws/spans` for `error_type` not null.

## Query templates (adjust field names after you look at real data)
CloudTrail logs in CloudWatch Logs:
```
fields eventTime, eventName, userIdentity.arn, errorCode, errorMessage, sourceIPAddress
| filter eventSource = "bedrock-agentcore.amazonaws.com"
| filter ispresent(errorCode)
| sort eventTime desc
```
Spans:
```
fields @timestamp, attributes.aws.operation.name, attributes.workload.identity.id, attributes.error_type
| filter ispresent(attributes.error_type)
| sort @timestamp desc
```
The span field paths are a guess. Check the real structure first (verify in your account).

## Related Systems Manager angle
The exam lists AWS Systems Manager among related technologies. A likely use is reading config or secrets from Parameter Store or running commands on self-hosted agent hosts. This is unconfirmed (verify in docs).
