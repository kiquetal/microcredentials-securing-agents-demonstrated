# Lab 1: Cognito + JWT authorizer + Runtime

Goal: deploy a runtime with a JWT authorizer, call it with a valid token, then break the call three ways and record the exact errors.
Time: about 65 minutes. Region used below: `us-east-1`. Source: [runtime-oauth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html). Commands were copied from that page. If one fails, read the page, because the CLI changes often.

## Prerequisites
- AWS CLI (latest), `jq`, `python3`, `uv`, Node.js 20.12 or later, and the AgentCore CLI: `npm install -g @aws/agentcore`, then `agentcore --version`. Setup details: [Get started with Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/agentcore-get-started-cli.html).
- Work inside `lab-workdir/` at the repo root (it is git-ignored). Never commit the project's `.env.local` or tokens.
- Your principal can create Cognito, IAM service-linked roles, and AgentCore runtimes. The creating role needs `iam:CreateServiceLinkedRole` for `runtime-identity.bedrock-agentcore.amazonaws.com`.
- Model access for the sample agent (Strands uses a Bedrock model by default). If the agent call fails on model access, enable the model in Bedrock. The auth tests still prove the authorizer works.

## Step 1: project
```bash
agentcore create --name OAuthAgentProject --template empty
cd OAuthAgentProject
```

## Step 2: Cognito pool, client, user, token
Save as `setup_cognito.sh`:
```bash
#!/bin/bash
export POOL_ID=$(aws cognito-idp create-user-pool --pool-name "MyUserPool" \
  --policies '{"PasswordPolicy":{"MinimumLength":8}}' --region $REGION | jq -r '.UserPool.Id')
export CLIENT_ID=$(aws cognito-idp create-user-pool-client --user-pool-id $POOL_ID \
  --client-name "MyClient" --no-generate-secret \
  --explicit-auth-flows "ALLOW_USER_PASSWORD_AUTH" "ALLOW_REFRESH_TOKEN_AUTH" \
  --region $REGION | jq -r '.UserPoolClient.ClientId')
aws cognito-idp admin-create-user --user-pool-id $POOL_ID --username $USERNAME \
  --region $REGION --message-action SUPPRESS > /dev/null
aws cognito-idp admin-set-user-password --user-pool-id $POOL_ID --username $USERNAME \
  --password $PASSWORD --region $REGION --permanent > /dev/null
export BEARER_TOKEN=$(aws cognito-idp initiate-auth --client-id "$CLIENT_ID" \
  --auth-flow USER_PASSWORD_AUTH --auth-parameters USERNAME=$USERNAME,PASSWORD=$PASSWORD \
  --region $REGION | jq -r '.AuthenticationResult.AccessToken')
echo "Pool id: $POOL_ID"
echo "Discovery URL: https://cognito-idp.$REGION.amazonaws.com/$POOL_ID/.well-known/openid-configuration"
echo "Client ID: $CLIENT_ID"
```
Run (do not print the token):
```bash
export REGION=us-east-1 USERNAME=labuser PASSWORD='ChangeMe-12345'
source setup_cognito.sh
```
Decode the token (see note 03) and fill the table:

| Claim | Value |
|-------|-------|
| `iss` | |
| `client_id` | |
| `aud` present? | |
| `scope` | |
| `token_use` | |

## Step 3: runtime with JWT authorizer
```bash
agentcore add runtime \
  --name OAuthAgent --language python --protocol HTTP \
  --authorizer-type CUSTOM_JWT \
  --authorizer-configuration \
  "{\"customJwtAuthorizer\":{\"discoveryUrl\":\"https://cognito-idp.$REGION.amazonaws.com/$POOL_ID/.well-known/openid-configuration\",\"allowedClients\":[\"$CLIENT_ID\"]}}" \
  --request-header-allowlist Authorization
cd app/OAuthAgent
uv init --bare
uv add "aws-opentelemetry-distro~=0.21.0" "bedrock-agentcore[strands-agents]>=1.18.1,<2" "botocore[crt]~=1.43.0" "strands-agents>=1.15,<2"
cd ../..
```
Create `app/OAuthAgent/main.py` exactly as in the runtime-oauth page (Step 4, "Create `app/OAuthAgent/main.py`"). Then:
```bash
agentcore build
agentcore deploy
agentcore status --json    # copy the runtime "arn"
```

## Step 4: invoke (the happy path)
```bash
export AGENT_ARN="<runtime arn>"
export ESCAPED_AGENT_ARN=$(echo -n "$AGENT_ARN" | jq -sRr '@uri')
export SESSION_ID=$(python3 -c "import uuid; print(uuid.uuid4())")
export INVOKE_URL="https://bedrock-agentcore.$REGION.amazonaws.com/runtimes/${ESCAPED_AGENT_ARN}/invocations?qualifier=DEFAULT"
export TOKEN=$BEARER_TOKEN
curl -s -i -X POST "$INVOKE_URL" \
  -H "Authorization: Bearer ${TOKEN}" -H "Content-Type: application/json" \
  -H "X-Amzn-Bedrock-AgentCore-Runtime-Session-Id: ${SESSION_ID}" \
  -d '{"prompt": "hello what is 1+1?"}'
```
Expect 200. The session ID header needs a long enough value. A UUID worked in the AWS example.

## Step 5: break it and record the exact response
Fill this table with the status code and body text you see. Use `curl -s -i` and keep the first 15 lines.

| Test | How | Status | Body / headers (short) | Cause in your words |
|------|-----|--------|------------------------|---------------------|
| A. No token | Drop the `Authorization` header | | | |
| B. Tampered token | Change one character in the middle of the token | | | |
| C. Wrong client | Create a second app client (`create-user-pool-client`), get a token from it, call the runtime | | | |
| D. Wrong audience | Redeploy with `allowedAudience:["not-my-api"]` added (config change, then `agentcore deploy`). Use the original token | | | |
| E. Wrong scope | Redeploy with `allowedScopes:["does.not.exist"]` | | | |
| F. Expired token | Optional: wait for expiry, or note the docs say tokens last 60 minutes by default | | | |

For D: the Cognito access token has no `aud`, so this tests the trap in note 03. Write down what happened. Then fix the config back to `allowedClients` only and confirm 200 again.

## Step 6 (optional, 10 minutes): read claims in the agent
Follow Step 7 of the runtime-oauth page to log the decoded claims (needs the `Authorization` header allowlist, which you already set). Look at the CloudWatch log group of the runtime.

## Review questions
1. Which single config field made test C fail?
2. Why does test D fail (or not) with an access token?
3. Which fields are checked together when you set more than one?

## Cleanup checklist
- [ ] `agentcore remove all` or the CLI's destroy/teardown for the project (check `agentcore --help`), then confirm the runtime is gone: `aws bedrock-agentcore-control list-agent-runtimes`
- [ ] Delete the Cognito pool: `aws cognito-idp delete-user-pool --user-pool-id $POOL_ID --region $REGION`
- [ ] Check ECR repo and CloudFormation stack created by the CLI are deleted (console check)
- [ ] Close the terminal or `unset TOKEN BEARER_TOKEN PASSWORD`
