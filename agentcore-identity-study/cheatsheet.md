# Cheat sheet (fill in as you study)

One page. Write answers from memory first, then check the notes. Prefilled lines come from verified docs. Empty ones are yours.

## 1. The four questions
| Question | Mechanism |
|----------|-----------|
| Who is the agent? | |
| Who may call it? | |
| What may it call? | |
| Who did what? | |

## 2. JWT authorizer fields to claims
| Field | Claim | Cognito access token has it? | Cognito ID token has it? |
|-------|-------|:-:|:-:|
| `discoveryUrl` | `iss` | | |
| `allowedClients` | `client_id` | | |
| `allowedAudience` | `aud` | | |
| `allowedScopes` | `scope` | | |
| custom claim | | | |

Rule: if more than one is set, ______ of them must pass. A runtime supports ______ or ______, not both.

## 3. Token APIs
| API | When | Verified? |
|-----|------|-----------|
| `GetWorkloadAccessToken` | | |
| `GetWorkloadAccessTokenForJWT` | | |
| `GetWorkloadAccessTokenForUserId` | | |

Who cannot call them directly: ______________________________

## 4. IAM snippets I must be able to write without looking
- Four ARNs for `GetResourceOauth2Token`:
  1. ______ 2. ______ 3. ______ 4. ______
- Deny the user-ID token path: ______________________________
- Deny the user-ID invoke header: ______________________________
- Tag condition for workload identities: ______________________________
- Runtime resource policy (Allow role X, Deny others): ______________________________
- Trust policy conditions for confused deputy: ______________________________

## 5. Commands
| Task | Command |
|------|---------|
| Create workload identity | `aws bedrock-agentcore-control create-workload-identity --name NAME` |
| List workload identities | `aws bedrock-agentcore-control list-workload-identities` |
| Delete workload identity | `aws bedrock-agentcore-control delete-workload-identity --name NAME` |
| Add tags | |
| Get workload token | |
| Cognito token | `aws cognito-idp initiate-auth --auth-flow USER_PASSWORD_AUTH ...` |
| Decode token | |
| Deploy JWT runtime | `agentcore add runtime ... --authorizer-type CUSTOM_JWT --authorizer-configuration '...'` then `agentcore build` and `agentcore deploy` |
| Look up CloudTrail events | |
| Create metric filter and alarm | |

## 6. Errors I have seen (fill from Labs 1 to 5)
| Status / message | Cause | Fix |
|------------------|-------|-----|
| | | |
| | | |
| | | |
| | | |

## 7. Detection
- Metric namespace: ______ Metric for token failures: ______
- Spans log group: ______ Application log `service_name`: ______
- CloudTrail events confirmed in my account: ______________________________
- Events NOT in CloudTrail (use another source): ______________________________

## 8. Exam-day order of work
1. Note region, account, resource names.
2. Read all tasks. Mark the easy ones.
3. Decode tokens before changing authorizers.
4. Test after each change. Record before and after status codes.
5. Leave 10 minutes to recheck tags and policies.

## 9. My slowest steps (from the dry run)
| Step | Time | Practice plan |
|------|------|---------------|
| | | |
