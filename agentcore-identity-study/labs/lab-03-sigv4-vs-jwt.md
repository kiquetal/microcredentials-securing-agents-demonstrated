# Lab 3: SigV4 path versus JWT path (objective 2)

Goal: run one runtime with IAM (SigV4) auth and one with JWT auth, scope each, and prove a runtime cannot use both.
Time: about 75 minutes. Sources: [runtime-oauth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html), [inbound-jwt-authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html), [resource-based-policies](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/resource-based-policies.html).

Reuse the project and Cognito pool from Lab 1 if you still have them. Rebuild them if you cleaned up.

## Part A: SigV4 runtime
1. Add a second runtime with the default IAM authorizer. Use `agentcore add runtime --help` to see the flag for the authorizer type (the Lab 1 command used `--authorizer-type CUSTOM_JWT`; check the allowed values for the IAM option). Deploy.
2. Invoke with a SigV4-signed request. Easiest with Python and boto3 (confirm the client and operation names in the SDK docs):
   ```python
   import boto3, json, uuid
   c = boto3.client("bedrock-agentcore", region_name="us-east-1")
   r = c.invoke_agent_runtime(
       agentRuntimeArn="<runtime arn>",
       runtimeSessionId=str(uuid.uuid4()),
       payload=json.dumps({"prompt": "hi"}).encode())
   print(r["ResponseMetadata"]["HTTPStatusCode"])
   ```
3. Make the IAM policy for the caller as tight as possible: only `bedrock-agentcore:InvokeAgentRuntime` on this runtime ARN.

## Part B: tests

| Test | Expected | Observed (status and message) |
|------|----------|-------------------------------|
| Caller role with the scoped Allow | success | |
| Caller with no `InvokeAgentRuntime` permission | AccessDenied | |
| Caller allowed on runtime X, calling runtime Y | AccessDenied | |
| Send a Cognito bearer token to the SigV4 runtime | rejected (record how) | |
| Send a SigV4 signed request to the JWT runtime from Lab 1 | rejected (record how) | |
| Same caller with the `X-Amzn-Bedrock-AgentCore-Runtime-User-Id` header but only `InvokeAgentRuntime` allowed | AccessDenied for `InvokeAgentRuntimeForUser` (verify) | |
| Add Deny `InvokeAgentRuntimeForUser` on `runtime/*` | still denied with header, normal invoke works | |

To send the user-ID header with curl you need SigV4 signing: `curl --aws-sigv4 "aws:amz:$REGION:bedrock-agentcore" --user "$AWS_ACCESS_KEY_ID:$AWS_SECRET_ACCESS_KEY" -H "x-amz-security-token: $AWS_SESSION_TOKEN" ...`. Adjust for the invoke URL from Lab 1. If this is awkward, use the SDK (look for a `runtimeUserId` parameter, which the docs mention in the CloudTrail guidance).

## Part C: resource-based policy
Pattern from the runtime-oauth page: Allow one role, Deny every other principal.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Sid": "AllowOnlyRoleX", "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCOUNT:role/ROLE_X" },
      "Action": "bedrock-agentcore:InvokeAgentRuntime",
      "Resource": "arn:aws:bedrock-agentcore:REGION:ACCOUNT:runtime/RUNTIME_ID" },
    { "Sid": "DenyOthers", "Effect": "Deny",
      "Principal": { "AWS": "*" },
      "Action": "bedrock-agentcore:InvokeAgentRuntime",
      "Resource": "arn:aws:bedrock-agentcore:REGION:ACCOUNT:runtime/RUNTIME_ID",
      "Condition": { "ArnNotEquals": { "aws:PrincipalArn": "arn:aws:iam::ACCOUNT:role/ROLE_X" } } }
  ]
}
```
Apply it with the AgentCore API for resource policies (find the command: `aws bedrock-agentcore-control help | grep -i resource-policy`). Test role X, another admin role, and root-like admin. Record who is blocked. Note that a Deny also blocks other roles in the same account.

## Part D: scoped JWT
On the JWT runtime from Lab 1, use more than client checking:
1. Add `allowedScopes` with a scope your token really has. Expect 200.
2. Add a custom claim rule. Example for a Cognito group (confirm the exact JSON shape in the [CustomJWTAuthorizerConfiguration API reference](https://docs.aws.amazon.com/bedrock-agentcore-control/latest/APIReference/API_CustomJWTAuthorizerConfiguration.html)): claim `cognito:groups`, type `STRING_ARRAY`, operator `CONTAINS`, value `agents`.
   - Create the group and add the user: `aws cognito-idp create-group` and `admin-add-user-to-group`.
   - Get a new token. Does it contain `cognito:groups`? Test with and without membership.
3. Record which combination of settings you used and what failed first.

## Part E: the "one authorizer type" rule
Try to update the JWT runtime to IAM auth (or the reverse) using the update API or a redeploy. Record whether it is allowed in place, and what the docs say: "An AgentCore Runtime can support either IAM SigV4 or JWT Bearer Token based inbound auth, but not both simultaneously."

## Cleanup checklist
- [ ] Both runtimes deleted
- [ ] Cognito group, user, pool deleted
- [ ] Test IAM roles and policies deleted
- [ ] Resource-based policy removed (it goes away with the runtime, confirm)
