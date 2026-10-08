Reference for the `cognito-auth` skill. The Terraform itself is
`templates/infrastructure/auth.tf.tmpl`, rendered into a new app as `infrastructure/aws/auth.tf`;
this page explains what it cannot.

# Terraform: the user pool, the client and email sending

**There is no dev/CI variant of this Terraform.** The pool you ship is the pool you would ship in
production: email verification required, no `lambda_config`, no `pre_sign_up` trigger, no
variable that can turn any of that off. The build proves sign-in with the per-deploy seeded
verifier account, which needs nothing in the pool but `ALLOW_USER_PASSWORD_AUTH` on the client
(see `references/verification.md`).

## What the rendered `auth.tf` holds

- `aws_cognito_user_pool.auth` — email as the username (case-insensitive), email verification by
  code, MFA off, account recovery by verified email, and this password policy:

  ```hcl
  password_policy {
    minimum_length    = 12 # ASVS 2.1.1
    require_lowercase = true
    require_uppercase = true
    require_numbers   = true
    require_symbols   = false
  }
  ```

  The platform's floor gate refuses fewer than eight characters, or lowercase, uppercase or
  numbers not required. The seeded demo and verifier passwords are 24 characters holding every
  class, so they meet any policy up to that.
- `aws_cognito_user_pool_client.web` — public (no secret), `ALLOW_USER_PASSWORD_AUTH`,
  `ALLOW_USER_SRP_AUTH`, `ALLOW_REFRESH_TOKEN_AUTH`, `prevent_user_existence_errors = "ENABLED"`,
  token revocation on, 60-minute ID and access tokens, 30-day refresh token.
- `aws_ssm_parameter.auth_config` — `/<api function name>/auth/config`, a plain JSON
  `{region, user_pool_id, client_id}` the API reads at cold start, and an
  `aws_iam_role_policy` letting the API's role read that one parameter. `lambda.tf` is the
  starter's, so the pool's ids reach the API this way rather than as environment variables.
- Outputs `auth_user_pool_id` and `auth_client_id`. The deploy workflow reads them, seeds the
  demo and verifier accounts into the pool under the deploy identity, and reports both on the
  deploy receipt. These names are the starter's auth seam (seam version 1); do not rename them.
- Variables `cognito_email_from` and `cognito_email_ses_identity_arn`, below.

## The pool has no usable email path until the account has a verified SES identity

Read this before calling the auth pack delivered. Sign-up cannot complete without the
emailed code, so a pool that cannot send mail to a stranger is a pool nobody outside
the deploy job can enrol in. **A delivered app with the `cognito_email_*` variables
unset, or set in an account without SES production access, has no usable email path at
all — not a low-volume one.** The visitor signs up, the app routes them to the
confirm-code screen, and no code ever arrives.

Two distinct failure modes produce that, and a new deployment usually starts in the
first and passes through the second:

1. **`cognito_email_from` and `cognito_email_ses_identity_arn` left empty.** The pool
   uses Cognito's own default sender, `COGNITO_DEFAULT` (`no-reply@verificationemail.com`).
   AWS documents that quota as **50 email messages sent daily per AWS account**,
   resetting at 09:00 UTC and **not adjustable**, and says of it: "For typical
   production environments, the default email limit is below the required delivery
   volume." The cap is per account, not per pool: every app deployed into the account
   shares it, and on 2026-09-10 three apps' verifiers exhausted one account's allowance
   between them and sign-up stopped for all three. Confirm the current figure on the
   [Cognito quotas
   page](https://docs.aws.amazon.com/cognito/latest/developerguide/quotas.html) before
   quoting it — it moves.
2. **Both variables set, in an account still in the SES sandbox.** A
   sandboxed account can send only to email addresses and domains already verified in
   that account, or to the SES mailbox simulator — at most 200 messages per 24 hours
   and 1 message per second. Every real visitor's address is unverified, so every real
   sign-up mails nobody. Sandbox status is per Region.

The default sender is therefore a **test** sender rather than a small production one,
and SES without production access reaches **only people already verified in the
account**. Neither state can confirm a stranger's sign-up, and nothing in this pack's
verification detects either — the build signs in as a seeded account and never sends mail
(`references/verification.md`).

## Sending through SES: the two variables the deployment must set

The rendered pool carries one `email_configuration` block that switches on two variables,
declared in `auth.tf` as below. The names are part of the pack's contract, so the deploy job and
the app's own docs can refer to them.

```hcl
variable "cognito_email_from" {
  description = "Verified From address for the user pool's email (e.g. \"no-reply@example.com\" or \"Example <no-reply@example.com>\"). Empty keeps Cognito's default sender."
  type        = string
  default     = ""
}

variable "cognito_email_ses_identity_arn" {
  description = "ARN of the verified SES identity (domain or address) in the same Region as the user pool, e.g. arn:aws:ses:eu-west-2:123456789012:identity/example.com. Empty keeps Cognito's default sender."
  type        = string
  default     = ""

  validation { # cross-variable reference: needs Terraform >= 1.9
    condition     = (var.cognito_email_ses_identity_arn == "") == (var.cognito_email_from == "")
    error_message = "Set cognito_email_from and cognito_email_ses_identity_arn together, or leave both empty."
  }
}

locals {
  # SES-backed sending is on only when both variables are set.
  cognito_ses_email = var.cognito_email_from != "" && var.cognito_email_ses_identity_arn != ""
}
```

With both set, the block resolves to `email_sending_account = "DEVELOPER"`,
`from_email_address = var.cognito_email_from`, `source_arn =
var.cognito_email_ses_identity_arn`. With either empty it resolves to
`email_sending_account = "COGNITO_DEFAULT"` and the two addresses are null — the same
pool the pack produced before the variables existed. **A production app must set both.**
The default sender is for a build's own smoke test and nothing else: its 50-a-day cap is
per AWS account, so every pool in the account draws on the same allowance, and one busy
pool's verifiers empty it for all of them.

Add `reply_to_email_address` to the block only if the app needs replies routed
somewhere; it is optional and the pack does not require a variable for it.

**What the pool sends, and what it must not.** Set the identity up, and describe it in
the delivered app's own documentation, as sending **account, two-factor authentication,
and account-recovery email only** — sign-up confirmation codes, MFA codes, and
forgotten-password codes. No marketing, no notifications, no mail the visitor did not
just trigger. That is the basis on which SES production access is requested for the
account (`--mail-type TRANSACTIONAL`), and the app's docs must say so plainly so that a
later change to send anything else is recognised as a change to that basis, not an
addition to a channel that already exists.

Terraform wires the block. It cannot create the two things the variables point at:

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
app's build, and they cannot be done from inside one. Once they are done, the two
variables are the only thing a deployment has to set.

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

## Never add a bypass

Earlier versions of this pack documented a var-gated `pre_sign_up` auto-confirm Lambda. **It was
removed and must not come back.** A trigger that confirms the harness's user confirms every
visitor's: with self-sign-up on, anyone could mint a confirmed account with `email_verified: true`
for an address they do not control. So the app's Terraform carries none of:

- an `auth_dev_auto_confirm` variable, or any similar flag;
- a `lambda_config` block on the pool, dynamic or otherwise;
- a `pre_sign_up` or `pre_authentication` trigger, count-gated or otherwise;
- a user-pool admin action (`cognito-idp:Admin*`) on the API's role.
