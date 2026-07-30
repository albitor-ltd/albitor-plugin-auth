# Auth pack plugin (`auth`)

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that packages the **default auth mechanism** for a delivered app: end-user **sign-up and login** backed by an Amazon Cognito user pool, with the UI rendered **in the app** and styled by whatever design-system skill the app chose.

It is one of the plugins Albitor makes available to its users, and the **default `auth-mechanism` capability** (ADR-0053 §3). It exists to fix a specific failure mode: auth improvised above a bare basic-auth floor produced a login-only app with no way to sign up. This skill makes end-user self-registration a deliberate, built-in outcome.

## What's inside

### Skill (loads automatically when relevant)

| Skill | Capability | Triggers on | Bundled references |
| --- | --- | --- | --- |
| `auth:cognito-auth` | `auth-mechanism` | Building or reviewing end-user authentication (sign-up + login) for a delivered app | Cognito Terraform (user pool + public app client), the in-app sign-up / login / confirm-code / resend screens, the JWT-validation API middleware + `GET /api/me`, and the skill-declared verification entry |

The skill provides:

- **Terraform** — a Cognito **user pool** + **public app client** (no secret), `USER_PASSWORD_AUTH` flow.
- **In-app UI** — sign-up, login, confirm-code, and resend screens, styled by the app's chosen design system, **not** the Cognito Hosted UI.
- **API** — JWT-validation middleware (JWKS signature + issuer/token-use/client/expiry checks) and a protected `GET /api/me`.
- **Email verification required in every environment** — secure by default: sign-ups must confirm an emailed code, so the confirm-code/resend screens are on the happy path. There is **no auto-confirm override in the delivered app**; Albitor's own build/verify loop self-proves signup by confirming its own throwaway user through the Cognito admin API from the deploy job, so the shipped user pool is identical to a production one.
- **Self-sign-up on by default** — the done-criterion is *"a first-time visitor can self-register from the app."*

### Capability contract

The skill's `SKILL.md` frontmatter declares `capability: auth-mechanism`, and the plugin `keywords` carry the matching `auth-mechanism` marker. This is how Albitor's plugin scanner classifies the skill as an auth-mechanism app-choice (ADR-0053 §7).

### Preview

`preview/index.html` is a self-contained, neutral **wireframe** of the four auth screens (sign-up, login, confirm-code, resend). It is a flow reference for scoping a build — **not** a style: the delivered app's screens are styled by its chosen design-system skill, so the wireframe deliberately uses a plain system-font layout and carries no design-system branding. It opens straight from `file://` with no build step.

## Installing

Add the marketplace that lists this plugin, then install:

```bash
claude plugin marketplace add albitor-ltd/albitor-plugins
claude plugin install auth@albitor-plugins
```

Or load it directly for one session during development:

```bash
claude --plugin-dir /path/to/albitor-plugin-auth
```

Validate the plugin structure:

```bash
claude plugin validate /path/to/albitor-plugin-auth
```

## Keeping it current

The skill's references are a distilled snapshot of how to wire Cognito, not a live mirror. The Terraform AWS provider and the AWS SDKs change, so the references always tell the assistant to confirm exact argument and command names against the live sources:

- Amazon Cognito developer guide — <https://docs.aws.amazon.com/cognito/latest/developerguide/>
- Terraform AWS provider (`aws_cognito_user_pool`, `aws_cognito_user_pool_client`) — <https://registry.terraform.io/providers/hashicorp/aws/latest/docs>
- AWS SDK for JavaScript v3 (`@aws-sdk/client-cognito-identity-provider`) — <https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/>

To refresh the references, re-distil from those sources into `skills/cognito-auth/references/` and bump the `version` in `.claude-plugin/plugin.json`.

## Licensing

All content in this plugin is original Albitor material licensed under MIT — see [`LICENCE`](LICENCE).
