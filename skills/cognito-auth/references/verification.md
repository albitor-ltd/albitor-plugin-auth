Reference for the `cognito-auth` skill: the done-criterion and the skill-declared verification entry
that the Albitor delivery factory reads.

# Verification: done-criterion and the skill-declared check

## Done-criterion

> **A first-time visitor can self-register from the app — reach sign-up from the sign-in page,
> create an account, and, once that account is confirmed, sign in and reach a protected route.**

The wording says *"once that account is confirmed"*, not *"confirm their email"*, on purpose:
nothing here drives the emailed code (see "What this pack does not prove").

## Two layers of check

1. **Universal `api-auth` security floor (Albitor-owned, blocking).** Declared non-public
   endpoints return `401` unauthenticated. The rendered `withIdentity` refuses every `/api/` route
   but `/api/health` and `/api/auth/*` without a verified token.
2. **Skill-declared check (this skill).** Sign-up is wired (the API calls Cognito `SignUp` and
   self-service sign-up is on), the `/sign-up` route renders its form, and it is reachable from
   the sign-in page through the "Sign up" link. The rendered `web/tests/auth-screens.spec.ts`
   proves the last two, signed out, without submitting anything.

## Signing in without an inbox: the seeded verifier

The build never registers to prove sign-in. On every deploy, `scripts/seed-demo-account.sh` (the
starter's, run by the deploy identity) creates two confirmed accounts in the pool with permanent
passwords and no email: the owner's demo account and the verifier account,
`verifier@albitor-verify.invalid`. Their passwords live in the app's own parameter store. A
journey walk signs in as the verifier with `USER_PASSWORD_AUTH` through the app's `/sign-in`
screen (`tests/auth.setup.ts`); the screenshot pass signs in as the demo account.

That needs four things of the rendered files, all true from the first commit:

- the `auth_user_pool_id` and `auth_client_id` outputs, which the seed and the walk read;
- `ALLOW_USER_PASSWORD_AUTH` on the client;
- no required second factor on the pool;
- the sign-in screen's test ids `sign-in-form`, `sign-in-email`, `sign-in-password` and
  `sign-in-button`, with `SIGN_IN_SCREEN_EXISTS = true` in `tests/auth.setup.ts`.

A build that changes any of them must keep the verifier able to sign in, or the walk fails.

## What this pack does not prove

- **The emailed confirmation code is never delivered or entered.** The seeded accounts are
  confirmed at creation. Verification proves the screens exist and that a confirmed account signs
  in and reaches a protected route; it does not prove Cognito can send mail from this pool.
- **The app's outbound email path is unexercised, and by default there is none.** With
  `cognito_email_from` and `cognito_email_ses_identity_arn` empty the pool uses Cognito's default
  sender, capped at 50 a day per AWS account; in an account still in the SES sandbox it reaches
  only addresses verified there. Either way a real visitor's sign-up cannot complete.
  `references/terraform.md` has the onboarding step that fixes it.

Record that gap in the delivery record and the README of any app handed over before the SES
identity exists.

## Machine-readable declaration

The block below is what a scanner ingests. Keep it in lockstep with the done-criterion and the
check above.

```yaml
albitor-skill-verification:
  capability: auth-mechanism
  skill: cognito-auth
  supports_self_signup: true
  self_signup_default: enabled              # self-service sign-up is on by default
  email_verification_default: required      # confirmation code required in every environment
  dev_self_prove_override: none             # the pool carries no verification bypass in any environment
  ci_self_prove_method: seeded-verifier-account # the deploy identity seeds a confirmed verifier; walks sign in as it with USER_PASSWORD_AUTH
  done_criterion: "A first-time visitor can self-register from the app — reach sign-up from the sign-in page, create an account, and, once that account is confirmed, sign in and reach a protected route."
  not_proven:
    - emailed-confirmation-code             # the seeded accounts are confirmed at creation; no code is delivered or entered
    - outbound-email-delivery               # COGNITO_DEFAULT / SES sending identity is never exercised
  requires_sending_identity: ses-identity-verified-in-the-app-account-with-production-access
  # Until that exists, a real visitor's sign-up cannot complete: no code arrives.
  # Onboarding work per AWS account (customer's own domain, customer's own account);
  # not a Terraform change and not detected by any check above.
  checks:
    - id: self-signup-wired-renders-and-reachable
      description: "Sign-up API path is wired, a sign-up route renders, and it is reachable from the sign-in page."
      proof:
        - api-path-wired          # POST /api/auth/sign-up calls Cognito SignUp; self-service sign-up enabled on the pool
        - signup-route-renders    # /sign-up serves the sign-up form
        - reachable-from-login    # the sign-in page links to /sign-up and the link resolves
      tier: presence              # presence + reachability, not the full sign-up journey
      blocking: false             # advisory skill check; the universal api-auth floor remains the blocking gate
      upgrade_path: playwright-signup-to-confirm-to-login-to-protected-route
  relies_on_floor:
    - id: api-auth
      description: "Declared non-public endpoints return 401 unauthenticated (Albitor-owned, blocking)."
```
