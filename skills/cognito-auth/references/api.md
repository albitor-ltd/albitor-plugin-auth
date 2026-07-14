Distilled reference for the `cognito-auth` skill. Verify JWKS/JWT library names against your stack's current crypto/JWT library before shipping.

# API: JWT-validation middleware + protected `GET /api/me`

The API is the gate. It trusts nothing the browser sends until it has **verified the token's signature** against the Cognito pool's JWKS and checked the standard claims. The Cognito Hosted UI is not involved; the browser obtained the token via `USER_PASSWORD_AUTH` (see `references/frontend.md`).

## What to verify on every protected request

The client sends `Authorization: Bearer <accessToken>`. The middleware must:

1. **Signature** — verify against the pool JWKS at
   `https://cognito-idp.<region>.amazonaws.com/<userPoolId>/.well-known/jwks.json`.
   Match the token's `kid` header to a JWKS key; cache the JWKS and refresh on an unknown `kid`.
2. **Issuer (`iss`)** — must equal `https://cognito-idp.<region>.amazonaws.com/<userPoolId>` (the `cognito_issuer` Terraform output).
3. **Token use (`token_use`)** — `access` for the access token (or `id` if you deliberately validate id tokens); reject the wrong kind.
4. **Client (`client_id`)** — must match the app client id.
5. **Expiry (`exp`)** — reject expired tokens.

Reject anything that fails with **`401 Unauthorized`** (no body detail that leaks why). This is exactly what the universal `api-auth` security floor probes for.

## Middleware sketch (Go)

```go
// Verify the bearer token against the Cognito pool JWKS and standard claims.
// Uses a JWKS-caching verifier keyed on the token's `kid`.
func RequireAuth(v *cognitoVerifier, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        raw, ok := bearerToken(r) // strips "Bearer " prefix
        if !ok {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        claims, err := v.Verify(r.Context(), raw) // signature + iss + token_use + client_id + exp
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), userKey{}, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

Use a maintained JWKS/JWT library for the verifier (e.g. `github.com/lestrrat-go/jwx` in Go, `aws-jwt-verify` in Node) rather than hand-rolling signature checks.

## Protected `GET /api/me`

The minimal protected endpoint that proves auth works end-to-end: it is reachable only with a valid token and echoes the caller's identity from the verified claims.

```go
mux.Handle("GET /api/me", RequireAuth(verifier, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    claims := r.Context().Value(userKey{}).(*Claims)
    writeJSON(w, http.StatusOK, map[string]any{
        "sub":   claims.Sub,
        "email": claims.Email,
    })
})))
```

- **Unauthenticated → 401.** With no/invalid token, `GET /api/me` returns `401`. This is the deterministic outcome the `api-auth` floor asserts.
- **Authenticated → 200** with the caller's `sub`/`email`.

## Relationship to the universal security floor

The Albitor `api-auth` floor is **always on and Albitor-owned**, independent of this skill: declared non-public endpoints must return `401` unauthenticated. This skill's job is to make sure the app *has* real auth wired so the floor passes for the right reason. Keep `GET /api/me` (and every non-public route) behind `RequireAuth`.

## Opt-in hardening: invite-only access

Self-sign-up is the default (see `SKILL.md`). To make an app invite-only instead, disable self-service sign-up on the user pool (admins create users via `AdminCreateUser`) and remove the in-app sign-up route — an explicit deviation from the default, not the out-of-the-box behaviour.
