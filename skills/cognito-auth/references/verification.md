Reference for the `cognito-auth` skill: the done-criterion and the skill-declared verification entry that the Albitor delivery factory reads.

# Verification: done-criterion and the skill-declared check

## Done-criterion

> **A first-time visitor can self-register from the app — reach sign-up from the login page, create an account, and, once that account is confirmed, sign in and reach a protected route.**

This is the single outcome the auth pack exists to guarantee. Email verification is
**required by default and in every environment** (secure by default), so a real user
confirms with the code Cognito emails them, on the in-app confirm-code screen. Build
the sign-up path first and treat it as the definition of done.

**State the criterion no more strongly than that.** The wording is deliberate: it says
*"once that account is confirmed"*, not *"confirm their email"*. Verification proves the
account lifecycle and the sign-in that follows it; it does **not** drive the emailed
code (see "What this pack does not prove" below). A criterion that claims the emailed
step would be a claim nothing in this pack tests — and a green check against it would
mean less than it appears to.

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

## Self-proving the journey with no inbox — and no bypass in the app

Albitor's autonomous build/verify loop has no inbox, so it cannot read an emailed
confirmation code. **That is a property of the harness, so it is solved in the
harness** — not by weakening the app.

Earlier versions of this pack solved it in the app, with a var-gated `pre_sign_up`
auto-confirm Lambda in the delivered Terraform. That has been **removed** (see
`references/terraform.md`): a trigger that confirms the harness's user confirms every
visitor's, so with self-sign-up on it let anyone mint a `CONFIRMED`,
`email_verified: true` account for an address they do not control. The capability is
unchanged; only its location moved.

**The deployed end-to-end spec now confirms its own throwaway user through the Cognito
admin API, from the deploy job.** The app's user pool is untouched by this — it carries
no trigger, no flag, and nothing that behaves differently in one environment than
another.

### Why the deploy job can do this

The deployed e2e step runs in the **same CI job** as `configure-aws-credentials`, after
`terraform apply`. Those credentials are still in scope — the same job already makes
AWS API calls with them (`aws ssm get-parameter` for app config, `aws s3 sync` and
`aws cloudfront create-invalidation` to ship the SPA). The pool id is already a
Terraform output (`cognito_user_pool_id`), read in the same job.

**Deploy-role permissions required:** `cognito-idp:AdminConfirmSignUp`,
`cognito-idp:AdminUpdateUserAttributes`, `cognito-idp:AdminDeleteUser`. A role that can
`terraform apply` a Cognito user pool is already identity-plane-capable, so this is
normally already held. Only a hand-written least-privilege deploy policy that
enumerates control-plane actions needs these three data-plane actions added.

### What the spec does

1. **Sign up through the UI**, as a real visitor would, with a throwaway address —
   e.g. `deployed-auth-${Date.now()}-w${workerIndex}@example.invalid`.
2. **Assert the app routes to the confirm-code screen.** Under the
   verification-required default this is the real behaviour, so assert it rather than
   skipping past it.
3. **Confirm the user out-of-band**, from the test process, with the job's AWS
   credentials.
4. **Log in through the UI** with the same credentials and **call a protected route**
   (`GET /api/me`) — the part the criterion is actually about.
5. **Delete the throwaway user**, in a `finally`/teardown so a failed assertion still
   cleans up. Leaving confirmed accounts behind in the delivered pool is its own small
   defect.

`AdminConfirmSignUp` confirms the account but does **not** mark the email verified, so
step 3 is two calls. Do both: an unverified email leaves `account_recovery_setting`'s
`verified_email` mechanism unusable, and the user is not in the state a real confirmed
user would be in.

```ts
// web/tests/deployed/auth-journey.spec.ts — confirm the harness's own user.
// Uses the deploy job's AWS credentials via the default provider chain; the app
// itself has no admin credentials and no auto-confirm affordance.
import {
  CognitoIdentityProviderClient,
  AdminConfirmSignUpCommand,
  AdminUpdateUserAttributesCommand,
  AdminDeleteUserCommand,
} from "@aws-sdk/client-cognito-identity-provider";

const admin = new CognitoIdentityProviderClient({ region: process.env.AWS_REGION });
const UserPoolId = process.env.COGNITO_USER_POOL_ID!; // from the terraform output

async function confirmTestUser(Username: string) {
  await admin.send(new AdminConfirmSignUpCommand({ UserPoolId, Username }));
  await admin.send(new AdminUpdateUserAttributesCommand({
    UserPoolId,
    Username,
    UserAttributes: [{ Name: "email_verified", Value: "true" }],
  }));
}

async function deleteTestUser(Username: string) {
  await admin.send(new AdminDeleteUserCommand({ UserPoolId, Username }));
}
```

The AWS CLI is equivalent if the spec would rather shell out — the CLI is already on
the GitHub-hosted runner:

```bash
aws cognito-idp admin-confirm-sign-up        --user-pool-id "$COGNITO_USER_POOL_ID" --username "$EMAIL"
aws cognito-idp admin-update-user-attributes --user-pool-id "$COGNITO_USER_POOL_ID" --username "$EMAIL" \
  --user-attributes Name=email_verified,Value=true
aws cognito-idp admin-delete-user            --user-pool-id "$COGNITO_USER_POOL_ID" --username "$EMAIL"
```

Pass `COGNITO_USER_POOL_ID` (from the `cognito_user_pool_id` Terraform output) into the
deployed-e2e step's `env:`, alongside the `DEPLOYED_BASE_URL` it already receives. The
deployed suite stays excluded from the ordinary CI e2e run (`testIgnore:
"**/deployed/**"` in `playwright.config.ts`); it runs post-apply, from the deploy job,
where the credentials are.

### What this buys

The **shipped user pool is byte-identical to a production pool.** There is no
browser-reachable bypass, no flag to copy into a production tfvars by mistake, and no
email pattern to get wrong on one side and not the other. A real human's sign-up now
fails **closed** (no code arrives, no account) rather than **open** (a confirmed
account for an unowned address).

## What this pack does not prove

Say this plainly rather than letting a green check imply otherwise:

- **The emailed confirmation code is never delivered or entered during verification.**
  Admin-confirm skips it, exactly as the removed override did. Verification proves that
  the confirm screen is reached and that a confirmed account can sign in — it does not
  prove Cognito can send mail from this pool, or that the code the user receives works.
- **The app's outbound email path is unexercised.** If the pool has no
  `email_configuration`, Cognito falls back to `COGNITO_DEFAULT`, which AWS documents as
  a low-volume test-only sender; a pool wired to SES in a sandboxed account can only
  mail verified identities. Neither condition is detected by anything here.

Proving the emailed step for real needs a mailbox the loop can read (an SES receipt
rule into S3, or a per-build throwaway inbox) and a working sending identity. That is
the upgrade path; nothing above is blocked while it is pending.

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
  email_verification_default: required      # DEFAULT: confirmation code required (secure by default)
  dev_self_prove_override: none             # REMOVED: the delivered pool carries no verification bypass in any environment
  ci_self_prove_method: admin-confirm-from-deploy-job # deployed e2e confirms its own throwaway user via the Cognito admin API with the deploy job's credentials
  done_criterion: "A first-time visitor can self-register from the app — reach sign-up from the login page, create an account, and, once that account is confirmed, sign in and reach a protected route."
  not_proven:
    - emailed-confirmation-code             # admin-confirm skips it; the code is never delivered or entered during verification
    - outbound-email-delivery               # COGNITO_DEFAULT / SES sending identity is never exercised
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
