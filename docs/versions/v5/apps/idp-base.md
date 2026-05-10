# Identity Provider (`swirlock-idp-base`)

Endpoint: `HTTP /oidc` (OIDC discovery, JWKS, auth, token, userinfo, end_session).

The Identity Provider is the **only** Swirlock ecosystem application that
speaks plain HTTP rather than the v5 ecosystem WebSocket envelope. This is a
deliberate exception, not a violation of the no-REST rule: OIDC is an
RFC-defined HTTP protocol (RFC 6749, OpenID Connect Core 1.0). Resource
servers (Chat Orchestrator, others) **do** continue to expose only their
ecosystem WebSocket — the IdP is consumed exclusively by clients during the
authentication redirect dance and by resource servers fetching JWKS to
validate tokens. No live request-path message ever traverses the IdP.

## Role

- Issues **JWT access tokens** (RS256, signed with the IdP's active signing key).
- Hosts the **registration** and **login** UI for end users.
- Owns the catalog of **client apps** (one row per app that consumes Swirlock auth).
- Owns the catalog of **end-user accounts**, scoped per client app.

## Endpoints

| Path | Method | Purpose |
| --- | --- | --- |
| `/oidc/.well-known/openid-configuration` | GET | Discovery document |
| `/oidc/jwks` | GET | Active public signing keys |
| `/oidc/auth` | GET | Authorization endpoint (Code + PKCE) |
| `/oidc/token` | POST | Token endpoint |
| `/oidc/me` | GET | UserInfo |
| `/oidc/session/end` | GET | RP-initiated logout |
| `/interaction/:uid` | GET | Login page (server-rendered) |
| `/interaction/:uid/login` | POST | Email + password submit |
| `/interaction/:uid/signup` | GET / POST | Registration form + submit |
| `/interaction/:uid/verify-email` | GET / POST | Code-entry page + submit |
| `/interaction/:uid/resend-code` | POST | Re-send verification code |
| `/interaction/:uid/confirm` | POST | Consent submit |
| `/interaction/:uid/abort` | POST | User-initiated denial |
| `/health` | GET | `{ "status": "ok" }` |

## Token format

Access tokens are RS256 JWTs. Required claims:

- `iss` — `${IDP_ISSUER}` (e.g. `http://127.0.0.1:3300/oidc` in dev).
- `aud` — the resource indicator the client requested (e.g. the Chat
  Orchestrator's URL). Resource servers MUST verify their own `aud`.
- `sub` — the IdP's internal account UUID. NOT the email. NOT the
  username.
- `client_id` — the consuming client app.
- `exp`, `iat`, `nbf` — standard.
- `scope` — space-separated scopes.

ID tokens follow OIDC standard claims (`sub`, `aud === client_id`, `nonce`).

## Account scoping

Accounts are scoped per `client_id`. The `accounts` table has
`UNIQUE(client_id, email)`: the same email can exist for two clients as two
distinct accounts, with independent passwords and verification states. An
account created for `swirlock-chatbot-ui` is **not** a valid identity for any
other client app — that client must run its own registration.

## Client registration

Clients are managed manually by the deployment owner, persisted in the IdP's
`clients` table. There is no public client-registration endpoint. The IdP
ships a CLI:

```
npm run client:add -- --id <client_id> --name "<Display name>" \
  --redirect-uri <url> [--redirect-uri <url>...] \
  --post-logout-redirect-uri <url> \
  --resource <orchestrator_url> \
  [--application-type web|native] \
  [--auth-method none|client_secret_basic]
npm run client:list
npm run client:remove -- --id <client_id>
```

Public SPAs (e.g. `swirlock-chatbot-ui`) register with
`token_endpoint_auth_method=none` and rely on PKCE for proof-of-possession.

## Resource server expectations

A resource server (Chat Orchestrator, future apps) validates the bearer token
on every connection by:

1. Decoding the JWT header, looking up the `kid` against a cached copy of
   `${IDP_ISSUER}/jwks`.
2. Refreshing the JWKS cache on `kid` miss.
3. Verifying signature with the matched key.
4. Verifying `iss === IDP_ISSUER`, `aud === own_resource_indicator`,
   `exp > now`, `nbf <= now`.
5. Reading `sub` as the user identity.

The IdP makes no claim about token revocation propagation. Access tokens are
valid until `exp`. Refresh tokens are revocable; revocation invalidates new
issuance, not in-flight tokens.

## Migration from v4

v5 introduces the IdP as a dedicated app. In v4 each service trusted a
shared static dev bearer token configured in its own
`service.config.cjs`. In v5 that path is removed: bearer tokens on the
WebSocket upgrade are IdP-issued JWTs, validated independently by each
resource server. Services migrating from v4 replace their bearer-token
comparison with JWT validation (see `API_CONVENTIONS.md` § Authentication).
