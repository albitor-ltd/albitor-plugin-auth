---
name: cognito-auth
description: Use when building or reviewing end-user authentication for a delivered web app that should let people sign up and sign in — the default auth mechanism. On a new BYOA app the sign-in is already in the repository, rendered from this pack's templates when the repository was created; the build extends it (authorisation, profile fields, copy, navigation, styling) rather than writing it. Covers the Cognito user pool and public client, the API's token check and GET /api/me, and the sign-in, sign-up, confirm-code and reset screens. Email verification is required, with no pre-sign-up trigger in any environment.
metadata:
  capability: auth-mechanism
---

# Cognito auth (default auth pack)

A delivered app gets real end-user authentication: a first-time visitor can **self-register from
the app**, confirm their email address, sign in, and reach protected API routes.

## Done-criterion

> **A first-time visitor can self-register from the app — reach sign-up from the sign-in page,
> create an account, and, once that account is confirmed, sign in and reach a protected route.**

## What is already in the repository

This pack ships its auth as **templated files** (`templates/`, described by
`templates/manifest.json`). The platform renders them into a new BYOA app's first commit, beside
the starter, so the app starts able to sign people in. Do not rebuild any of this: read it, then
extend it.

| File | What it does |
| --- | --- |
| `infrastructure/aws/auth.tf` | The user pool (email sign-in, verification required, 12-character policy, no MFA, no trigger) and a public client with `ALLOW_USER_PASSWORD_AUTH`, `ALLOW_USER_SRP_AUTH` and `ALLOW_REFRESH_TOKEN_AUTH`. Outputs `auth_user_pool_id` and `auth_client_id`. Writes the API's auth config parameter and lets the API's role read it. Declares `cognito_email_from` and `cognito_email_ses_identity_arn`. |
| `api/server/identity.go` | The identity seam: `withIdentity` refuses every `/api/` request without a verified token except `/api/health` and `/api/auth/*`, serves `GET /api/me`, and puts the token's `sub` on the request; `ownerFromRequest` returns it. |
| `api/server/auth_token.go` | The RS256 check against the pool's JWKS: `iss`, `token_use`, client, expiry. |
| `api/server/auth_cognito.go` | `POST /api/auth/{sign-in,sign-up,confirm,resend,forgot,reset,sign-out}`. The API calls Cognito's public operations and keeps the session in two HttpOnly, Secure, SameSite=Strict cookies, refreshing an expired one. |
| `web/public/index.html`, `auth.js`, `auth.css` | The screens at `/sign-in`, `/sign-up`, `/confirm` and `/reset`, cross-linked, with stable `data-testid`s. `auth.js` sends a signed-out visitor to `/sign-in` and back again. |
| `web/serve.mjs` | Serves the shell for a route path, as CloudFront does, so `/sign-in` works in the pre-ship verify. |
| `web/tests/auth.setup.ts`, `auth-screens.spec.ts` | `SIGN_IN_SCREEN_EXISTS = true`, and a signed-out spec that the screens render and link to each other. |
| `scripts/seed-journey-fixtures.sh` | Seeds the journey fixtures as the verifier and as the demo account, signing in through the app's own API. |

The API makes the sign-in calls, not the browser, for three reasons: the page's CSP allows
`connect-src 'self'` only; the web app has no build step to carry the pool's ids; and HttpOnly
cookies keep every token away from page scripts.

**An app created before its pack version shipped templates, or on another lane, has none of
this.** Build the same shape by hand, with this pack's `templates/` as the reference.

## What the build still writes

- **Authorisation.** Who owns which record, groups mapped to permissions, and which routes are
  public under the app's posture. Every user-owned store call already takes `ownerFromRequest`.
- **Profile attributes** beyond email, and any sign-up fields the brief asks for.
- **Where the screens sit** in the app's navigation, and their styling through the app's design
  system. Restyle freely, but keep the `data-testid` values: `auth.setup.ts`, the journey specs and
  the verifier target them.
- **The app's own copy**, including the verification email's wording (`auth.tf`).
- **Anything the brief adds** — MFA, social sign-in, invite-only, domain restriction — as an edit
  to the rendered files.

## Rules

- **Do not weaken the pool.** No `lambda_config`, no `pre_sign_up` trigger, no auto-confirm flag:
  not count-gated, not dev-only. The floor gates refuse all of them.
- **No user-pool admin action on the API's role.** The demo and verifier accounts are seeded by
  the deploy identity (`scripts/seed-demo-account.sh`), never by the function serving requests.
- **Keep the password policy at or above the template's** (ASVS 2.1.1: at least 12 characters).
- **Keep what the verifier needs**: the `auth_user_pool_id` and `auth_client_id` outputs,
  `ALLOW_USER_PASSWORD_AUTH` on the client, and no required second factor. A brief that asks for
  MFA uses `OPTIONAL`, so the seeded verifier still signs in.
- **Never take the owner from the request body or a query parameter.**
- **No Cognito Hosted UI.** The screens are the app's own.
- **Only one spec registers.** A sign-up sends an email, and the default sender allows 50 a day
  per AWS account. Other specs use the seeded session `auth.setup.ts` creates.

## Email: no real visitor can confirm until the account can send

Until the app's AWS account holds a verified SES identity with production access, and
`cognito_email_from` and `cognito_email_ses_identity_arn` are set, a real visitor who signs up
gets no code and cannot confirm. Say so in the delivery record of any app handed over in that
state. `references/terraform.md` has the two failure modes and the onboarding step.

## References

| File | When to read it |
| --- | --- |
| `references/terraform.md` | Email sending (SES identity, production access, the two variables), and why the pool has no dev/CI variant. |
| `references/verification.md` | The done-criterion, how the build proves sign-in without an inbox, and the machine-readable verification block. |

## Verification (skill-declared)

The universal `api-auth` floor still applies: declared non-public endpoints return `401`
unauthenticated. This skill's own check is that sign-up is wired, its route renders, and it is
reachable from the sign-in page. `references/verification.md` carries the machine-readable
`albitor-skill-verification` block.
