Reference for the `cognito-auth` skill: the done-criterion and the skill-declared verification entry that the Albitor delivery factory reads.

# Verification: done-criterion and the skill-declared check

## Done-criterion

> **A first-time visitor can self-register from the app.**

This is the single outcome the auth pack exists to guarantee. Build the sign-up path first and treat it as the definition of done.

## Two layers of check

1. **Universal `api-auth` security floor (Albitor-owned, unchanged, blocking).**
   Declared non-public endpoints must return `401` when called unauthenticated.
   This is not declared here — it applies to every auth mechanism and catches a
   *broken* auth outcome whatever the cause. See `references/api.md`.

2. **Skill-declared check (this skill).**
   The self-sign-up proof for **v1** is: **the sign-up API path is wired *and* a
   sign-up route renders** — deliberately *not* a full browser end-to-end test.
   - *Wired:* the app calls Cognito `SignUp` (self-service sign-up is enabled on
     the pool) and a new user can subsequently authenticate.
   - *Renders:* the app serves a sign-up route/page that returns its form.
   - **Residual gap (accepted for v1):** a rendered-but-dead-end sign-up form
     would pass this presence-level proof. The upgrade path is to promote it to
     the Playwright verification tier (full browser sign-up → login → protected
     route), not to add heavier assertions here.

## Machine-readable declaration

The block below is the unambiguous, greppable form of the skill's verification
intent — this is what a scanner ingests. Keep it in lockstep with the
done-criterion and the check described above. (The exact machine-ingestion
schema is finalised separately; the keys here state the intent explicitly and
stably.)

```yaml
albitor-skill-verification:
  capability: auth-mechanism
  skill: cognito-auth
  supports_self_signup: true
  self_signup_default: enabled
  done_criterion: "A first-time visitor can self-register from the app."
  checks:
    - id: self-signup-wired-and-renders
      description: "Sign-up API path is wired and a sign-up route renders."
      proof:
        - api-path-wired      # app calls Cognito SignUp; self-service sign-up enabled on the pool
        - signup-route-renders # app serves a sign-up page that returns its form
      tier: presence          # v1: NOT a full browser e2e
      blocking: false         # advisory skill check; the universal api-auth floor remains the blocking gate
      upgrade_path: playwright-signup-to-login-to-protected-route
  relies_on_floor:
    - id: api-auth
      description: "Declared non-public endpoints return 401 unauthenticated (Albitor-owned, blocking)."
```
