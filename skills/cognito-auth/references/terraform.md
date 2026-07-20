Distilled reference for the `cognito-auth` skill. Amazon Cognito changes; verify exact argument names and defaults against the Terraform AWS provider docs (`aws_cognito_user_pool`, `aws_cognito_user_pool_client`) and the Cognito developer guide before shipping.

# Terraform: Cognito user pool + public app client

The identity provider for the auth pack. Two required resources — a **user pool** (the directory of accounts) and a **public app client** (how the browser talks to it) — plus a **dev-only** auto-confirm Lambda that is gated off outside development environments.

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

  # DEFAULT: real email verification — Cognito emails a confirmation code and the
  # user confirms via the in-app confirm-code screen. No auto-confirm Lambda here.
  # The dev/CI auto-confirm override wires a pre_sign_up Lambda into this block;
  # see "Dev/CI override" below. Production/default leaves lambda_config unset.

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

By default, sign-ups require a **verified email**. Cognito emails a confirmation
code on sign-up; the user enters it on the in-app confirm-code screen (see
`references/frontend.md`) before they can log in. This is **secure by default** —
no unverified accounts, no auto-confirm Lambda. Nothing extra is needed beyond
`auto_verified_attributes = ["email"]` on the pool (above) and leaving
`lambda_config` unset. Configure the message via `verification_message_template`
and, for production volumes, wire `email_configuration` to SES.

```hcl
# In aws_cognito_user_pool.app — customise the emailed code message (optional):
verification_message_template {
  default_email_option = "CONFIRM_WITH_CODE"
  email_subject        = "Your ${var.app_name} verification code"
  email_message        = "Your verification code is {####}"
}
```

## Dev/CI override: auto-confirm (development only)

Albitor's own autonomous build/verify loop and design-system reviews must be able
to self-prove the sign-up path **without a real inbox**. For that, and only in a
**development or CI environment**, wire an auto-confirm pre-sign-up Lambda that
confirms the user and marks the email verified with no emailed code. This is a
**dev-only override** — it must never be enabled in production or in the default
configuration, because it defeats email verification.

The trigger is a **Go Lambda on the `provided.al2023` runtime** — the same OS-only
runtime and prebuilt-`bootstrap` packaging the app's API Lambda already uses, so
there is no managed-language runtime to age out. (This override previously ran on
`nodejs20.x`; it was moved to `provided.al2023` to stop the recurring managed-runtime
EOL churn.) CI already sets up Go and builds the API `bootstrap`; build this one the
same way — see "Building the bootstrap" below.

Gate it on an explicit variable that defaults to *off* so the override cannot
leak into a real deployment:

```hcl
variable "auth_dev_auto_confirm" {
  description = "DEV/CI ONLY — auto-confirm sign-ups with no emailed code so the build can self-prove signup without a real inbox. MUST stay false in production/default."
  type        = bool
  default     = false # secure by default: real email verification
}

# Count-gated so the Lambda, its zip, and its wiring only exist in dev/CI.
# archive_file zips the prebuilt bootstrap binary (built in CI — see "Building the
# bootstrap" below); count-gating it too means a disabled build never needs the
# binary present, so terraform validate/plan stay green with the override off.
data "archive_file" "auto_confirm" {
  count       = var.auth_dev_auto_confirm ? 1 : 0
  type        = "zip"
  source_file = "${path.module}/lambda-src/auto-confirm/bootstrap"
  output_path = "${path.module}/build/auto-confirm.zip"
}

resource "aws_lambda_function" "auto_confirm" {
  count            = var.auth_dev_auto_confirm ? 1 : 0
  function_name    = "${var.app_name}-cognito-auto-confirm-DEVONLY"
  runtime          = "provided.al2023" # OS-only runtime — no managed-language EOL
  handler          = "bootstrap"
  architectures    = ["x86_64"]        # match the API Lambda's go build target
  role             = aws_iam_role.auto_confirm[0].arn
  filename         = data.archive_file.auto_confirm[0].output_path
  source_code_hash = data.archive_file.auto_confirm[0].output_base64sha256
}

resource "aws_lambda_permission" "cognito_invoke" {
  count         = var.auth_dev_auto_confirm ? 1 : 0
  statement_id  = "AllowCognitoInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.auto_confirm[0].function_name
  principal     = "cognito-idp.amazonaws.com"
  source_arn    = aws_cognito_user_pool.app.arn
}
```

Attach the trigger to the pool **only** when the override is on. Keep the pool's
base config verification-required; add the `pre_sign_up` trigger conditionally so
the default deployment never carries it:

```hcl
# Add to aws_cognito_user_pool.app with a dynamic block so it appears only in dev/CI:
dynamic "lambda_config" {
  for_each = var.auth_dev_auto_confirm ? [1] : []
  content {
    pre_sign_up = aws_lambda_function.auto_confirm[0].arn
  }
}
```

```go
// lambda-src/auto-confirm/main.go — DEV/CI ONLY pre-sign-up trigger: confirm
// every new user, skip the emailed code. Never deploy this outside development.
package main

import (
	"context"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func handler(_ context.Context, event events.CognitoEventUserPoolsPreSignup) (events.CognitoEventUserPoolsPreSignup, error) {
	event.Response.AutoConfirmUser = true
	event.Response.AutoVerifyEmail = true // marks email verified without a code
	return event, nil
}

func main() {
	lambda.Start(handler)
}
```

```
// lambda-src/auto-confirm/go.mod
module autoconfirm

go 1.25

require github.com/aws/aws-lambda-go v1.49.0
```

### Building the bootstrap

The `bootstrap` binary is built in CI **before** `terraform plan`/`apply`, exactly
like the API Lambda — Terraform only zips the prebuilt artifact, it never compiles.
With the app's existing `actions/setup-go` step in place, add a build for this
module (only needed when the override is on):

```bash
cd infrastructure/aws/lambda-src/auto-confirm
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -tags lambda.norpc -o bootstrap .
```

Git-ignore the built binary (`infrastructure/aws/lambda-src/auto-confirm/bootstrap`)
and `infrastructure/aws/build/` — they are CI artifacts, like `api/bootstrap`.
Unit-test the handler with `go test ./...` (this replaces the old Node `node --test`
step). `terraform validate` stays green without a build because the count-gated
`archive_file` isn't read when `auth_dev_auto_confirm` is false.

Set `auth_dev_auto_confirm = true` only in the dev/CI tfvars so Albitor's
build/verify loop can register and log in a throwaway user end-to-end; production
and the default leave it `false` and exercise the real confirmation-code flow.

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
