# Lab 2: Self-hosted workload identity + tag-based access (objective 1)

Goal: create a workload identity for a self-hosted agent, fetch a workload access token, and restrict access with tags.
Time: about 90 minutes. Sources: [creating-agent-identities](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/creating-agent-identities.html), [identity-tagging](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/identity-tagging.html), [get-workload-access-token](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/get-workload-access-token.html), [scope-credential-provider-access](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/scope-credential-provider-access.html).

Self-hosted here means your laptop or an EC2 instance with an IAM role. A laptop with an assumed role is enough.

## Part A: create identities
```bash
export REGION=us-east-1
aws bedrock-agentcore-control create-workload-identity --name lab-agent-a --region $REGION
aws bedrock-agentcore-control list-workload-identities --region $REGION
```
Record the `workloadIdentityArn`.

Tags on create. The CLI reference lists a tags option. Check the exact syntax first:
```bash
aws bedrock-agentcore-control create-workload-identity help | grep -i -A3 tags
```
Then create `lab-agent-b` with `Team=blue` and re-create or tag `lab-agent-a` with `Team=red` (use `tag-resource` if it exists: `aws bedrock-agentcore-control tag-resource help`).

Verify tags: `aws bedrock-agentcore-control list-tags-for-resource --resource-arn <arn>` (verify the command name with `help`).

## Part B: workload access token
Two IAM roles with a trust policy for your own principal (or use two sets of credentials):
- `LabAgentRoleA`: allow `bedrock-agentcore:GetWorkloadAccessToken` on the directory and `lab-agent-a` ARNs only.
- `LabAgentRoleB`: the same for `lab-agent-b`.

Directory and identity ARNs:
```
arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default
arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/lab-agent-a
```
Fetch a token (CLI name to confirm: `aws bedrock-agentcore get-workload-access-token help`):
```bash
aws bedrock-agentcore get-workload-access-token --workload-name lab-agent-a --region $REGION
```
Or Python:
```python
from bedrock_agentcore.services.identity import IdentityClient
c = IdentityClient("us-east-1")
print(bool(c.get_workload_access_token(workload_name="lab-agent-a")))   # do not print the token
```
Do not print or store the token.

## Part C: tests

| Test | Expected | Observed (error text) |
|------|----------|-----------------------|
| Role A fetches token for `lab-agent-a` | success | |
| Role A fetches token for `lab-agent-b` | AccessDenied | |
| Role A with `GetWorkloadAccessTokenForUserId` (user_id="u1") but only `GetWorkloadAccessToken` allowed | AccessDenied | |
| Add the ForUserId action to Role A, call again | success | |
| Add explicit Deny for `GetWorkloadAccessTokenForUserId` on `*`, call again | AccessDenied despite Allow | |
| Call it against a **Runtime-managed** identity (from Lab 1, if you kept it) | "WorkloadIdentity is linked to a service..." | |

## Part D: tag-based restriction
Goal: one role can use only workload identities whose `Team` tag equals its own principal tag.
1. Tag `LabAgentRoleA` with `Team=red`, and `LabAgentRoleB` with `Team=blue` (`aws iam tag-role`).
2. Replace the per-ARN allow with a wildcard on identities plus a condition (pattern from note 02):
   ```json
   {
     "Effect": "Allow",
     "Action": ["bedrock-agentcore:GetWorkloadAccessToken"],
     "Resource": [
       "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default",
       "arn:aws:bedrock-agentcore:REGION:ACCOUNT:workload-identity-directory/default/workload-identity/*"
     ],
     "Condition": { "StringEquals": { "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}" } }
   }
   ```
3. Test both roles against both identities. Record results.

Questions to answer from the observed behavior. Do not assume:
- Did the condition apply to the directory ARN too? If the call failed, which resource caused it?
- Does the action support `aws:ResourceTag`? Check the [Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonbedrockagentcore.html). If it does not, use a different gate (Resource ARN naming, a permissions boundary, or tags on the principal only).
- Can the role change the tag? Add a Deny for `bedrock-agentcore:TagResource`/`UntagResource` and test.

## Part E (optional): credential provider scoping
Create an API key credential provider (`aws bedrock-agentcore-control create-api-key-credential-provider help`). Write the four-resource policy from note 02 for `lab-agent-a`, and a Deny for another provider.

## Cleanup checklist
- [ ] `aws bedrock-agentcore-control delete-workload-identity --name lab-agent-a` and `lab-agent-b`
- [ ] Delete the API key provider if you created one
- [ ] Delete `LabAgentRoleA` and `LabAgentRoleB` and their policies
- [ ] `list-workload-identities` shows none of the lab names
