---
name: cognito-auth
description: Use when building or reviewing end-user authentication for a delivered web app that should let people sign up and log in — the default auth mechanism. Covers an Amazon Cognito user pool with a public app client (USER_PASSWORD_AUTH), in-app sign-up / login / confirm-code / resend UI styled by the app's own design system (not the Cognito Hosted UI), JWT-validation API middleware, and a protected GET /api/me. Self-sign-up is supported and on by default; email verification is required by default (secure by default), with a dev/CI-only auto-confirm override so the build can self-prove signup without a real inbox.
metadata:
  capability: auth-mechanism
---

# Cognito auth (default auth pack)

Give a delivered app real end-user authentication: a first-time visitor can **self-register from the app**, log in, and reach protected API routes. This is the default `auth-mechanism` skill — reach for Cognito with in-app sign-up and login unless the app deliberately picked another mechanism.

It exists because the alternative — improvising auth above a bare basic-auth floor — produced a login-only app with no way to sign up. This skill makes end-user sign-up a deliberate, built-in outcome.

## Done-criterion (the thing that must be true)

> **A first-time visitor can self-register from the app — sign up, confirm their email, and sign in.**

If a new user cannot create an account from the running app's own UI, confirm their email, and then sign in, the auth pack is not done — no matter how much of the wiring is in place. Build the sign-up path first, not last.

Self-registration must be **reachable**, not merely present: a first-time visitor who lands on the login page must be able to get to sign-up from the UI itself. The login and sign-up screens **must cross-link** — a "Don't have an account? Sign up" link on login and an "Already have an account? Log in" link on sign-up (see `references/frontend.md`). A sign-up form that only exists at a URL nobody can navigate to does not satisfy the done-criterion.

## Self-sign-up support (declared)

This skill **supports self-sign-up, and it is enabled by default.** Anyone who reaches the app can create their own account through the in-app sign-up form; there is no admin-invite step in the default configuration. Self-sign-up handling and its parameters (required email verification vs. the dev/CI auto-confirm override, allowed email domains) belong to *this* skill — there is no global app-wide sign-up switch to keep in sync across auth mechanisms. If the app needs invite-only access, that is an explicit hardening on top of this skill (see `references/api.md`), not the default.

## What this skill delivers

Four moving parts, each detailed in a bundled reference:

1. **Terraform — the identity provider.** A Cognito **user pool** plus a **public app client** (no client secret — the app is a browser SPA and cannot keep one) with the `USER_PASSWORD_AUTH` flow enabled. See `references/terraform.md`.
2. **In-app UI — the screens.** Sign-up, login, confirm-code, and resend-code flows, rendered **in the app and styled by the app's chosen design-system skill** — *not* the Cognito Hosted UI. See `references/frontend.md`.
3. **API — the gate.** JWT-validation middleware that verifies Cognito-issued access tokens against the pool's JWKS, plus a protected `GET /api/me` that echoes the caller's identity. See `references/api.md`.
4. **Verification — the proof.** The done-criterion above, plus the machine-readable skill-declared check the delivery factory reads. See `references/verification.md` and the [Verification](#verification-skill-declared) section below.

## Email verification: required by default

Sign-ups **require a verified email by default** — secure by default. Cognito emails a confirmation code; the user enters it on the in-app confirm-code screen before they can log in. The confirm-code and resend screens are therefore on the **default happy path**, not an opt-in extra.

Albitor's own autonomous build/verify loop has no inbox, so a **dev/CI-only auto-confirm override** (a var-gated pre-sign-up Lambda, off by default) lets the build register and log in a throwaway user with no emailed code — while production and the default stay verification-required. The override must never be enabled outside a development environment. `references/terraform.md` shows the verification-required default and the clearly dev-only override.

## Styling comes from the design system, never the Hosted UI

Do **not** enable or link to the **Cognito Hosted UI**. The sign-up and login screens are pages *in the app*, built from the components of whatever design-system skill the app selected (GDS, shadcn, MUI, Carbon, …). Consume that skill for the form fields, buttons, error summary, and layout — this skill owns the *auth behaviour and flow*, the design-system skill owns the *look*. If no design-system skill is loaded, ask which one applies before hand-rolling form styling.

## Bundled references — read the relevant one before building

| File | When to read it |
| --- | --- |
| `references/terraform.md` | Provisioning the user pool + public app client, the `USER_PASSWORD_AUTH` flow, the verification-required default vs. the dev/CI auto-confirm override, and the outputs the app needs. |
| `references/frontend.md` | Building the sign-up / login / confirm-code / resend screens, calling Cognito from the browser, storing and refreshing tokens, and wiring the styling to the chosen design system. |
| `references/api.md` | The JWT-validation middleware (JWKS verification, issuer/audience/expiry checks), the protected `GET /api/me`, and how the universal `api-auth` security floor sees this. |
| `references/verification.md` | The done-criterion and the machine-readable skill-declared verification entry (the v1 self-sign-up proof). |

## Verification (skill-declared)

The universal `api-auth` security floor still applies and is unchanged: declared non-public endpoints must return `401` when called unauthenticated. That is Albitor-owned and blocking whatever auth mechanism is chosen — it catches a *broken* auth outcome.

On top of the floor, this skill declares its own check. **For v1 the self-sign-up proof is: the sign-up API path is wired, a sign-up route renders, *and* that route is reachable from the login page — a login→sign-up link exists and resolves to the sign-up route — not a full browser end-to-end test.** The reachability clause closes the earlier dead-end gap: a sign-up form that renders only at an un-navigable URL no longer passes. (Residual gap accepted for v1: the check confirms the link resolves, not that a full sign-up→confirm→login journey succeeds; the upgrade path is to promote it to the Playwright verification tier.)

The machine-readable form of this declaration — the intent a scanner ingests — lives in `references/verification.md` under the `albitor-skill-verification` block. Keep the done-criterion and the check in lockstep with that block.

## Working rules

- **Public app client, no secret.** A browser SPA cannot keep a client secret; generating one breaks `USER_PASSWORD_AUTH` from the browser. Never add a secret to the app client.
- **Never ship credentials in code.** The user pool id and app client id are public config (safe in the browser); they reach the app as build-time config, not hardcoded secrets. There is no secret to embed.
- **Build sign-up first.** The done-criterion is self-registration. Start there, then login, then the protected route — not the other way round.
- **Cross-link login and sign-up.** Self-registration must be reachable from the UI, not just present at a URL: the login screen carries a "Don't have an account? Sign up" link and the sign-up screen an "Already have an account? Log in" link, both built from the design system's link/anchor component. These are in-app route links — they do **not** contradict the Hosted-UI prohibition above, which forbids linking *out* to the Cognito Hosted UI.
- **Both verification screens, always.** Confirm-code and resend are on the default happy path (verification required); build them as first-class screens. The dev/CI auto-confirm override is the only thing that skips them, and only in development.
- **Validate tokens server-side.** Never trust a token the browser hands you without verifying its signature against the pool JWKS and checking issuer, use/audience, and expiry. See `references/api.md`.
- **Let the design system style it.** Consume the app's design-system skill for every form control and error pattern; this skill does not carry its own visual style.

## Related

- The app's chosen **design-system skill** (`capability: design-system`) — owns the look of every auth screen.
- The universal `api-auth` security floor — Albitor-owned, always on, blocks a broken auth outcome regardless of mechanism.
