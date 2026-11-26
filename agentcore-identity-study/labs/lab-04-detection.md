# Lab 4: Detection with CloudTrail and CloudWatch (objective 3)

Goal: find which identity events are visible, build alarms on them, and trigger them on purpose.
Time: about 75 minutes. Read note 04 first. Many facts here are unconfirmed, so this lab is mostly discovery.

Prerequisite: identities and runtimes from Labs 1 to 3, or recreate one workload identity and one JWT runtime quickly.

## Part A: enable observability on identity resources
Source: [observability-identity-metrics](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-identity-metrics.html) and [Add observability to your resources](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/observability-configure.html).
1. Enable CloudWatch Transaction Search if the docs ask for it (verify on the get-started page).
2. Enable observability on the workload identity (console on the AgentCore Identity page, or the API shown in the configure page).
3. Generate some calls: successful and failed token fetches from Lab 2.
4. Wait a few minutes and list metrics:
   ```bash
   aws cloudwatch list-metrics --namespace AWS/Bedrock-AgentCore --region $REGION
   ```
   Record which metric names and dimensions actually appear.

## Part B: CloudTrail discovery
1. Create (or use) a trail that delivers to CloudWatch Logs. Include management events. Check the advanced event selector resource types for AgentCore (console, "Data events", resource type list).
2. Repeat these actions, then look them up:
   - create and delete a workload identity
   - tag a workload identity
   - a successful and a denied token fetch
   - a successful and a denied `InvokeAgentRuntime`
   - an `InvokeAgentRuntime` with the user-ID header
3. Query:
   ```bash
   aws cloudtrail lookup-events --region $REGION \
     --lookup-attributes AttributeKey=EventSource,AttributeValue=bedrock-agentcore.amazonaws.com --max-results 50 \
     | jq -r '.Events[] | [.EventTime, .EventName, .Username] | @tsv'
   ```
   The event source name might differ (for example `bedrock-agentcore.amazonaws.com` is a guess). If nothing returns, look up by `EventName` for `CreateWorkloadIdentity`.
4. Fill in the discovery table in note 04.

## Part C: alarms
Build these and trigger each one:

| # | Detects | Mechanism | Threshold | Triggered how | Fired? |
|---|---------|-----------|-----------|---------------|--------|
| 1 | Token fetch failures | CloudWatch alarm on `AWS/Bedrock-AgentCore` `WorkloadAccessTokenFetchFailures`, Sum, 1 min | >= 1 | Role with no permission calls the token API | |
| 2 | Use of the user-ID path | CloudTrail logs metric filter or EventBridge rule on `GetWorkloadAccessTokenForUserId` (confirm event name) | >= 1 | Call it once | |
| 3 | Workload identity created or deleted | Same | >= 1 | Create/delete one | |
| 4 | Denied invokes | Metric filter on `errorCode` containing `AccessDenied` for the AgentCore event source | >= 1 | Denied invoke from Lab 3 | |

Metric filter example (adjust names after Part B):
```bash
aws logs put-metric-filter --log-group-name <cloudtrail-log-group> \
  --filter-name agentcore-userid-token \
  --filter-pattern '{ $.eventSource = "bedrock-agentcore.amazonaws.com" && $.eventName = "GetWorkloadAccessTokenForUserId" }' \
  --metric-transformations metricName=UserIdTokenCalls,metricNamespace=LabSecurity,metricValue=1
aws cloudwatch put-metric-alarm --alarm-name lab-userid-token \
  --namespace LabSecurity --metric-name UserIdTokenCalls --statistic Sum --period 60 \
  --evaluation-periods 1 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching --region $REGION
```
Add an SNS action only if you want mail. Keep `--treat-missing-data notBreaching` so the alarm stays OK when there are no events.

## Part D: Logs Insights
Run the two query templates from note 04 against CloudTrail and `aws/spans`. Fix the field names to match what you see, and save the working queries in `cheatsheet.md`.

## Review questions
1. Which detections worked from metrics alone, and which needed CloudTrail?
2. Which events were missing from CloudTrail? What would you use instead?
3. What is the fastest way to answer "which principal fetched a token for which user ID in the last hour"?

## Cleanup checklist
- [ ] Delete alarms and metric filters
- [ ] Stop or delete the trail if you created it for this lab (keep data events off to avoid cost)
- [ ] Delete the CloudWatch log group and the S3 bucket if only used here
- [ ] Disable observability settings you changed
