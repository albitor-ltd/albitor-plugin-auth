Reference for the `cognito-auth` skill: the done-criterion and the skill-declared verification entry that the Albitor delivery factory reads.

# Verification: done-criterion and the skill-declared check

## Done-criterion

> **A first-time visitor can self-register from the app — sign up, confirm their email, and sign in.**

This is the single outcome the auth pack exists to guarantee. Email verification
is **required by default** (secure by default), so self-registration includes the
confirmation-code step. Build the sign-up path first and treat it as the
definition of done.

## Two layers of check

1. **Universal `api-auth` security floor (Albitor-owned, unchanged, blocking).**
   Declared non-public endpoints must return `401` when called unauthenticated.
   This is not declared here — it applies to every auth mechanism and catches a
   *broken* auth outcome whatever the cause. See `references/api.md`.

2. **Skill-declared check (this skill).**
   The self-sign-up proof for **v1** is: **the sign-up API path is wired, a
   sign-up route renders, *and* that route is reachable from the login page** —
   deliberately *not* a full browser end-to-end test.
   - *Wired:* the app calls Cognito `SignUp` (self-service sign-up is enabled on
     the pool) and a new user can subsequently authenticate.
   - *Renders:* the app serves a sign-up route/page that returns its form.
   - *Reachable:* the login page carries a link to the sign-up route (the
     "Don't have an account? Sign up" cross-link), and that link resolves to the
     sign-up route — a first-time visitor on the login page can get to sign-up
     from the UI, not just by typing the URL.
   - **Gap closed vs. earlier v1:** a rendered-but-dead-end sign-up form (present
     at a URL but not linked from login) previously passed; the reachability
     clause now fails it. **Residual gap (accepted for v1):** the check confirms
     the login→sign-up link resolves, not that the full sign-up → confirm →
     login → protected-route journey succeeds. The upgrade path is to promote it
     to the Playwright verification tier for that full journey, not to add
     heavier assertions here.

**Self-provability under required verification.** Email verification is required
by default, so a real emailed code would normally be needed to confirm a new
user. Albitor's autonomous build/verify loop has no inbox, so the build's **dev
environment** uses the documented **dev/CI auto-confirm override** (see
`references/terraform.md`) to confirm the throwaway test user with no emailed
code. Production and the default stay verification-required; only the dev
environment carries the override.

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
  self_signup_default: enabled              # self-service sign-up is on by default
  email_verification_default: required      # DEFAULT: confirmation code required (secure by default; was auto-confirm)
  dev_self_prove_override: auto-confirm-dev-only # dev/CI-only auto-confirm so the build proves signup without a real inbox (see terraform.md)
  done_criterion: "A first-time visitor can self-register from the app — sign up, confirm their email, and sign in."
  checks:
    - id: self-signup-wired-renders-and-reachable
      description: "Sign-up API path is wired, a sign-up route renders, and it is reachable from the login page."
      proof:
        - api-path-wired          # app calls Cognito SignUp; self-service sign-up enabled on the pool
        - signup-route-renders    # app serves a sign-up page that returns its form
        - reachable-from-login    # the login page links to the sign-up route and the link resolves (closes the dead-end-form gap)
      tier: presence              # v1: NOT a full browser e2e (presence + reachability, not the full journey)
      blocking: false             # advisory skill check; the universal api-auth floor remains the blocking gate
      upgrade_path: playwright-signup-to-confirm-to-login-to-protected-route
  relies_on_floor:
    - id: api-auth
      description: "Declared non-public endpoints return 401 unauthenticated (Albitor-owned, blocking)."
```
