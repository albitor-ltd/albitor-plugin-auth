Distilled reference for the `cognito-auth` skill. Amazon Cognito changes; verify exact argument names and defaults against the Terraform AWS provider docs (`aws_cognito_user_pool`, `aws_cognito_user_pool_client`) and the Cognito developer guide before shipping.

# Terraform: Cognito user pool + public app client

The identity provider for the auth pack. Two resources, and only two — a **user pool** (the directory of accounts) and a **public app client** (how the browser talks to it).

**There is no dev/CI variant of this Terraform.** The pool you ship is the pool you would ship in production: email verification required, no `lambda_config`, no `pre_sign_up` trigger, no variable that can turn any of that off. Albitor's build/verify loop still proves the sign-up journey end to end — it does it by confirming its own throwaway user through the Cognito **admin API from the deploy job**, which needs nothing in the app's infrastructure. See `references/verification.md`.

## What "public app client" means and why it matters

The app is a browser SPA. A browser cannot keep a client secret, so the app client is created **without a secret** (`generate_secret = false`, the default) and uses the `USER_PASSWORD_AUTH` auth flow. If you generate a secret, `USER_PASSWORD_AUTH` from the browser fails with `SECRET_HASH` errors — so never add one for the SPA client.

## User pool

```hcl
resource "aws_cognito_user_pool" "app" {
  name = "${var.app_name}-users"

  # Sign in with an email address.
  username_attributes      = ["email"]
  auto_verified_attributes = ["email"] # DEFAULT: email must be verified by a confirmation code

  password_policy {
    minimum_length    = 8
    require_lowercase = true
    require_numbers   = true
    require_uppercase = true
    require_symbols   = false
  }

  schema {
    name                = "email"
    attribute_data_type = "String"
    required            = true
    mutable             = true
  }

  # Real email verification, in EVERY environment — Cognito emails a confirmation
  # code and the user confirms via the in-app confirm-code screen. Deliberately no
  # lambda_config and no pre_sign_up trigger: a pre-sign-up trigger that
  # auto-confirms would defeat email verification for every visitor, not just the
  # test harness. CI confirms its own test user out-of-band — see verification.md.

  account_recovery_setting {
    recovery_mechanism {
      name     = "verified_email"
      priority = 1
    }
  }
}
```

## Public app client

```hcl
resource "aws_cognito_user_pool_client" "spa" {
  name         = "${var.app_name}-spa"
  user_pool_id = aws_cognito_user_pool.app.id

  generate_secret = false # public client — browser SPA, no secret

  explicit_auth_flows = [
    "ALLOW_USER_PASSWORD_AUTH", # login with username + password
    "ALLOW_REFRESH_TOKEN_AUTH", # silent token refresh
  ]

  # Hosted UI is NOT used — the app renders its own sign-up/login screens.
  # No callback_urls / allowed_oauth_flows needed for USER_PASSWORD_AUTH.

  access_token_validity  = 60  # minutes
  id_token_validity      = 60  # minutes
  refresh_token_validity = 30  # days
  token_validity_units {
    access_token  = "minutes"
    id_token      = "minutes"
    refresh_token = "days"
  }

  prevent_user_existence_errors = "ENABLED"
}
```

## Email verification (the default sign-up experience)

Sign-ups require a **verified email**, in every environment. Cognito emails a
confirmation code on sign-up; the user enters it on the in-app confirm-code screen
(see `references/frontend.md`) before they can log in. This is **secure by
default and by construction** — no unverified accounts, no auto-confirm Lambda, no
flag that turns it off. Nothing extra is needed beyond
`auto_verified_attributes = ["email"]` on the pool (above) and leaving
`lambda_config` unset. Customise the message with
`verification_message_template`.

```hcl
# In aws_cognito_user_pool.app — customise the emailed code message (optional):
verification_message_template {
  default_email_option = "CONFIRM_WITH_CODE"
  email_subject        = "Your ${var.app_name} verification code"
  email_message        = "Your verification code is {####}"
}
```

## The pool has no usable email path until the account has a verified SES identity

Read this before calling the auth pack delivered. Sign-up cannot complete without the
emailed code, so a pool that cannot send mail to a stranger is a pool nobody outside
the deploy job can enrol in. **A delivered app with no `email_configuration` and no SES
production access has no usable email path at all — not a low-volume one.** The visitor
signs up, the app routes them to the confirm-code screen, and no code ever arrives.

Two distinct failure modes produce that, and a new deployment usually starts in the
first and passes through the second:

1. **No `email_configuration` on the pool.** Cognito falls back to its own default
   sender, `COGNITO_DEFAULT` (`no-reply@verificationemail.com`). AWS documents that
   quota as **50 email messages sent daily per AWS account**, resetting at 09:00 UTC
   and **not adjustable**, and says of it: "For typical production environments, the
   default email limit is below the required delivery volume." Confirm the current
   figure on the [Cognito quotas
   page](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html) before
   quoting it — it moves.
2. **`email_configuration` wired to SES, in an account still in the SES sandbox.** A
   sandboxed account can send only to email addresses and domains already verified in
   that account, or to the SES mailbox simulator — at most 200 messages per 24 hours
   and 1 message per second. Every real visitor's address is unverified, so every real
   sign-up mails nobody. Sandbox status is per Region.

The default sender is therefore a **test** sender rather than a small production one,
and SES without production access reaches **only people already verified in the
account**. Neither state can confirm a stranger's sign-up, and nothing in this pack's
verification detects either — the deployed e2e confirms its own user through the admin
API and never sends mail (`references/verification.md`).

## The `email_configuration` block, and what the deployment must supply

```hcl
# In aws_cognito_user_pool.app — send through the account's own SES identity.
email_configuration {
  email_sending_account  = "DEVELOPER"             # this account's SES, not COGNITO_DEFAULT
  source_arn             = var.ses_identity_arn    # arn:aws:ses:<region>:<account>:identity/<domain>
  from_email_address     = var.auth_from_email     # e.g. "Example <no-reply@example.com>"
  reply_to_email_address = var.auth_reply_to_email # optional
}
```

Terraform wires the block. It cannot create the two things the block depends on:

- **A verified SES identity in the app's own AWS account.** The account's owner
  verifies a domain (or a single address) by publishing the DNS records SES issues. The
  identity must sit in a Region the pool is allowed to send from — check the user pool
  Region against the [SES Region table for Cognito
  identities](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-email.html),
  as not every Cognito Region can send from its own.
- **SES production access for that account and Region.** Requested from the SES console
  ("Request production access") or with `aws sesv2 put-account-details
  --production-access-enabled --mail-type TRANSACTIONAL …`, then reviewed and granted by
  AWS Support. **It is a support request, not a Terraform change**; no `terraform apply`
  substitutes for it, and a plan is green whether or not it has been granted.

One further prerequisite is IAM, not SES: setting `email_sending_account = "DEVELOPER"`
makes Cognito create a service-linked role, so whoever applies the Terraform needs
`iam:CreateServiceLinkedRole`.

Both SES steps are **onboarding work for the AWS account**, done once per account and
per Region, before or alongside the first delivery into it. They are not part of an
app's build, and they cannot be done from inside one.

## Whose domain sends

The sending domain is **the customer's own**, verified in the customer's own AWS
account — the account the app is deployed into — with SES production access requested
for that account at onboarding. Albitor does not send on a delivered app's behalf and
lends it no sending identity: mail from the app is the customer's mail, under the
customer's domain and the customer's sending reputation, which is also what keeps one
customer's bounce rate off another's. An Albitor-operated sending identity, with
safeguarding checks on message content, belongs only to apps Albitor itself hosts —
a separate lane, and not the one that delivers an app into a customer's account. This
is decided; the per-deployment question is *which* domain, never whose.

## No dev/CI bypass — deliberately

Earlier versions of this pack documented a var-gated `pre_sign_up` auto-confirm
Lambda (`auth_dev_auto_confirm`) so the build could register a throwaway user with
no emailed code. **It has been removed, and it must not be reintroduced.**

The affordance was legitimate; its location was not. Albitor's loop needs to confirm
*one throwaway user*; it does not follow that the **shipped user pool** should carry
a trigger that confirms *everybody*. With self-sign-up on by default and the pool's
`account_recovery_setting` trusting `verified_email`, an auto-confirm trigger lets
anyone on the internet mint a `CONFIRMED` account with `email_verified: true` for an
address they do not control — identity forgery, in the one environment the app is
actually delivered in. It also removes the honest failure mode: with the trigger a
bogus sign-up fails *open*, without it a sign-up with no reachable email path simply
fails *closed*.

So do not add any of the following to the app's Terraform:

- a `variable "auth_dev_auto_confirm"` (or any similarly named flag);
- a `lambda_config` / `dynamic "lambda_config"` block on `aws_cognito_user_pool.app`;
- a `pre_sign_up` (or `pre_authentication`) trigger, count-gated or otherwise;
- a `lambda-src/auto-confirm` module, its `archive_file`, IAM role, log group, invoke
  permission, or CI build step.

The test harness confirms its own user instead, from outside the app, using
credentials CI already holds — `references/verification.md` has the exact steps. The
only thing the app's Terraform has to provide for that is the
`cognito_user_pool_id` output, which it already does (below).

## Outputs the app needs

These are **public** values — safe to expose to the browser as build-time config. There is no secret to protect.

```hcl
output "cognito_user_pool_id" { value = aws_cognito_user_pool.app.id }
output "cognito_app_client_id" { value = aws_cognito_user_pool_client.spa.id }
output "cognito_region" { value = var.aws_region }
# Issuer/JWKS the API middleware verifies against:
output "cognito_issuer" {
  value = "https://cognito-idp.${var.aws_region}.amazonaws.com/${aws_cognito_user_pool.app.id}"
}
```

The API validates tokens against `${issuer}/.well-known/jwks.json` — see `references/api.md`.
