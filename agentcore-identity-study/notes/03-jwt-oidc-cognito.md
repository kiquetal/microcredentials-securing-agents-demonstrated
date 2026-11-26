# 03. JWT, OIDC and Cognito

This is your weakest area, so spend the time here. Sources: [runtime-oauth](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-oauth.html), [inbound-jwt-authorizer](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/inbound-jwt-authorizer.html).

## JWT structure
`header.payload.signature`, each part base64url-encoded.
- **Header:** `alg` (for example RS256) and `kid` (which key signed it).
- **Payload (claims):** the data.
- **Signature:** verified with the IdP public key (JWKS). AgentCore finds the keys through the discovery URL.

## Claims you must recognize
| Claim | Meaning | AgentCore authorizer field |
|-------|---------|----------------------------|
| `iss` | Issuer URL | `discoveryUrl` (its `issuer` must equal `iss`) |
| `aud` | Intended API/resource | `allowedAudience` |
| `client_id` | App that requested the token | `allowedClients` |
| `scope` | Space-separated permissions | `allowedScopes` |
| `sub` | User or subject ID | Used with `iss` as the user identity for the token vault |
| `exp`, `iat` | Expiry and issue time (Unix seconds) | Checked automatically |
| `token_use` | Cognito: `access` or `id` | Use the **access** token for the authorizer |
| `cognito:groups` | Cognito group membership | Candidate for a custom claim rule (`STRING_ARRAY` with `CONTAINS`) |

## Cognito specifics (the common trap)
- A Cognito **access token** has `client_id` and no `aud`. A Cognito **ID token** has `aud` and no `client_id`. So:
  - Access token: configure `allowedClients`.
  - ID token: configure `allowedAudience`.
  - The runtime-oauth page shows an access token payload with `client_id`, `scope`, `token_use: access`.
  - Mixing these is the classic cause of 401 (verify the behavior in Lab 1).
- Default token validity is 60 minutes in the doc's setup.
- The default scope seen in the doc's decoded token is `aws.cognito.signin.user.admin`. Custom scopes need a resource server in the user pool (verify in the Cognito docs).

## OIDC discovery
- Discovery URL for Cognito: `https://cognito-idp.REGION.amazonaws.com/POOL_ID/.well-known/openid-configuration`
- The JSON at that URL contains `issuer` and `jwks_uri`. The `issuer` must equal the token `iss`. A typo in the pool ID or region breaks everything.
- Pattern the authorizer enforces: `^[^\s?#]+/\.well-known/(openid-configuration|oauth-authorization-server)(/[^\s?#]+)?$`. No query string, no fragment.

## Get and decode a token (shell)
After running the Cognito setup in Lab 1:
```bash
# get a token
export TOKEN=$(aws cognito-idp initiate-auth --client-id "$CLIENT_ID" \
  --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME="$USERNAME",PASSWORD="$PASSWORD" \
  --region "$REGION" | jq -r '.AuthenticationResult.AccessToken')

# decode the payload (portable version)
echo "$TOKEN" | cut -d '.' -f2 | tr '_-' '/+' | { read p; printf '%s%s' "$p" "$(printf '=%.0s' $(seq 1 $(( (4 - ${#p} % 4) % 4 )) ))"; } | base64 -d 2>/dev/null | jq
```
The AWS doc uses `base64 -D` (macOS). On Linux use `base64 -d` as above. Both the shell version and this Python alternative were tested locally against a fake token:
```bash
python3 -c 'import sys,base64,json;p=sys.argv[1].split(".")[1];print(json.dumps(json.loads(base64.urlsafe_b64decode(p+"="*(-len(p)%4))),indent=2))' "$TOKEN"
```

Check the discovery document:
```bash
curl -s "https://cognito-idp.$REGION.amazonaws.com/$POOL_ID/.well-known/openid-configuration" | jq '{issuer, jwks_uri}'
```

## How the authorizer decides (checklist)
1. Is there a `Bearer` token? If not: 401 with `WWW-Authenticate`.
2. Is the signature valid for the keys at `jwks_uri`, and is `iss` the discovery issuer, and is `exp` in the future?
3. If `allowedClients` is set: does `client_id` match?
4. If `allowedAudience` is set: does `aud` contain a match?
5. If `allowedScopes` is set: does `scope` contain at least one match?
6. If custom claims are set: does each rule match?
All configured checks must pass.

## Common 401 causes
| Cause | How to confirm |
|-------|----------------|
| Wrong `allowedClients` value or it is an ID token | Decode the token. Compare `client_id` / `aud` |
| `allowedAudience` set but access token has no `aud` | Decode. No `aud` present |
| Issuer mismatch (pool ID or region) | Compare `iss` with `issuer` from the discovery URL |
| Token expired | `exp` is in the past. Get a new token |
| Scope missing | Compare `scope` with `allowedScopes` |
| Custom claim rule fails | Compare claim name, type and operator |
| Token sent to a SigV4 runtime (or the other way round) | Check the runtime's authorizer type |

Exact HTTP status and message for each case: record them in Lab 1. Do not assume them.

## Video search terms
"JWT explained", "OIDC discovery document", "Cognito access token vs ID token".
