Distilled reference for the `cognito-auth` skill. Amazon Cognito changes; verify exact argument names and defaults against the Terraform AWS provider docs (`aws_cognito_user_pool`, `aws_cognito_user_pool_client`) and the Cognito developer guide before shipping.

# Terraform: Cognito user pool + public app client

The identity provider for the auth pack. Two required resources — a **user pool** (the directory of accounts) and a **public app client** (how the browser talks to it) — plus an optional auto-confirm Lambda that is the default sign-up experience.

## What "public app client" means and why it matters

The app is a browser SPA. A browser cannot keep a client secret, so the app client is created **without a secret** (`generate_secret = false`, the default) and uses the `USER_PASSWORD_AUTH` auth flow. If you generate a secret, `USER_PASSWORD_AUTH` from the browser fails with `SECRET_HASH` errors — so never add one for the SPA client.

## User pool

```hcl
resource "aws_cognito_user_pool" "app" {
  name = "${var.app_name}-users"

  # Sign in with an email address.
  username_attributes      = ["email"]
  auto_verified_attributes = ["email"] # relevant only when real verification is on (see below)

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

  # DEFAULT: auto-confirm sign-ups (no emailed code). See "Auto-confirm" below.
  lambda_config {
    pre_sign_up = aws_lambda_function.auto_confirm.arn
  }

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

## Auto-confirm (the default sign-up experience)

By default, sign-ups are **auto-confirmed** with a tiny pre-sign-up Lambda so no verification email is sent and the user can log in immediately. This is the frictionless POC default and makes the security floor's checks deterministic.

```hcl
resource "aws_lambda_function" "auto_confirm" {
  function_name = "${var.app_name}-cognito-auto-confirm"
  runtime       = "nodejs20.x"
  handler       = "index.handler"
  role          = aws_iam_role.auto_confirm.arn
  filename      = data.archive_file.auto_confirm.output_path
}

resource "aws_lambda_permission" "cognito_invoke" {
  statement_id  = "AllowCognitoInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.auto_confirm.function_name
  principal     = "cognito-idp.amazonaws.com"
  source_arn    = aws_cognito_user_pool.app.arn
}
```

```js
// index.js — pre-sign-up trigger: confirm every new user, skip email verification.
export const handler = async (event) => {
  event.response.autoConfirmUser = true;
  event.response.autoVerifyEmail = true; // marks email verified without a code
  return event;
};
```

## Opt-in hardening: real email verification

To require a genuine emailed code instead of auto-confirming, make a **one-attribute switch**: remove the `pre_sign_up` Lambda from `lambda_config` (or make it a no-op) and let Cognito's default flow send a verification code to `auto_verified_attributes = ["email"]`. The confirm-code and resend screens the app already ships (see `references/frontend.md`) then carry the real code entry — no UI rework. Configure the message via `verification_message_template` and, for production volumes, wire `email_configuration` to SES.

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
