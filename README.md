# Auth pack plugin (`auth`)

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) that packages the **default auth mechanism** for a delivered app: end-user **sign-up and login** backed by an Amazon Cognito user pool, with the UI rendered **in the app** and styled by whatever design-system skill the app chose.

It is one of the plugins Albitor makes available to its users, and the **default `auth-mechanism` capability** (ADR-0053 §3). It exists to fix a specific failure mode: auth improvised above a bare basic-auth floor produced a login-only app with no way to sign up. This skill makes end-user self-registration a deliberate, built-in outcome.

## What's inside

### Templates (rendered into a new app's first commit)

`templates/manifest.json` describes the files this pack renders into a new BYOA `web_app`
repository, beside the starter, when the repository is created (Albitor ADR-0088). The manifest
names the capability (`auth-mechanism`), provider (`cognito`), skill (`cognito-auth`), archetype
(`web_app`), lane (`byoa`), the starter auth-seam version it implements (`1`) and its machine
sign-in (`password`). Templates are plain files: the platform substitutes only its own
`{{albitor.<token>}}` tokens, and these templates use none.

| Repository path | What it is |
| --- | --- |
| `infrastructure/aws/auth.tf` | User pool (email verification, 12-character password policy, no trigger, no MFA), public client (`ALLOW_USER_PASSWORD_AUTH`, `ALLOW_USER_SRP_AUTH`, `ALLOW_REFRESH_TOKEN_AUTH`), the API's auth config parameter, and the `auth_user_pool_id` / `auth_client_id` outputs |
| `api/server/identity.go`, `auth_token.go`, `auth_cognito.go`, `identity_test.go` | The API identity seam: JWKS-verified tokens, `/api/*` refused without one except `/api/health`, `GET /api/me`, `sub` as the record owner, and `POST /api/auth/*` with HttpOnly session cookies |
| `web/public/index.html`, `auth.js`, `auth.css` | Sign-in, sign-up, confirm-code and reset screens with stable `data-testid`s |
| `web/serve.mjs` | The local static server, serving the shell for route paths as CloudFront does |
| `web/tests/auth.setup.ts`, `web/tests/auth-screens.spec.ts` | `SIGN_IN_SCREEN_EXISTS = true`, and a signed-out spec of the screens |
| `web/tests/local-sign-in.spec.ts` | Two test identities signed in through the starter's local issuer are two subjects; skips outside a local run with it |
| `scripts/seed-journey-fixtures.sh` | Journey fixtures seeded as the verifier and the demo account |

Every pack version's rendered tree is gated at ingest by the platform's floor checks, so a build
starts from a tree that passes. A build may then change any rendered file; the same gates run on
what it ships.

#### A local run trusts the starter's local issuer, and nothing else does

The starter ships a local issuer (`api/internal/localauth`): with `ALBITOR_LOCAL=1
ALBITOR_LOCAL_AUTH=1` the API starts an in-process stand-in for the pool that mints Cognito-shaped
tokens for the test identities `alice`, `bob` and `admin`, so specs sign in without sign-up,
e-mail or a call to Cognito. `auth_cognito.go`'s `authSourceFromEnv` trusts it only in that case:
a named pool (`AUTH_USER_POOL_ID`/`AUTH_CLIENT_ID`) or a deployed function
(`AWS_LAMBDA_FUNCTION_NAME`, set by the Lambda runtime) is decided first and never trusts it, and
the issuer itself refuses to start in either. In a local run the sign-in calls (`POST
/api/auth/*`) answer 501 `local`. The issuer's URL, key path and client id are its contract,
restated in `auth_cognito.go` so the pack does not import the starter's package.

### Skill (loads automatically when relevant)

| Skill | Capability | Triggers on | Bundled references |
| --- | --- | --- | --- |
| `auth:cognito-auth` | `auth-mechanism` | Building or reviewing end-user authentication (sign-up and sign-in) for a delivered app | Email sending and the no-bypass rule (`terraform.md`), and the skill-declared verification entry (`verification.md`) |

The skill tells a build what is already in the repository and what it still writes:
authorisation, profile fields, the screens' place and styling in the app, its copy, and anything
the brief adds.

- **Email verification is required in every environment.** There is no auto-confirm trigger in
  the delivered pool, and the build proves sign-in with the per-deploy seeded verifier account
  rather than by registering.
- **No email path until the app's AWS account has one.** Until that account holds a verified
  Amazon SES identity with production access, a real visitor's sign-up cannot complete. The pool
  takes `cognito_email_from` and `cognito_email_ses_identity_arn`; both set switches it to
  `DEVELOPER` sending through that identity. See `skills/cognito-auth/references/terraform.md`.
- **Self-sign-up is on by default.** The done-criterion is *"a first-time visitor can
  self-register from the app."*

### Capability contract

The skill's `SKILL.md` frontmatter declares `capability: auth-mechanism`, and the plugin `keywords` carry the matching `auth-mechanism` marker. This is how Albitor's plugin scanner classifies the skill as an auth-mechanism app-choice (ADR-0053 §7).

### Preview

`preview/index.html` is a self-contained, neutral **wireframe** of the auth screens. It is a flow reference for scoping a build, not a style. It opens straight from `file://` with no build step.

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

The templates and references are a snapshot of how to wire Cognito, not a live mirror. The Terraform AWS provider and Cognito change; confirm argument and operation names against the live sources:

- Amazon Cognito developer guide — <https://docs.aws.amazon.com/cognito/latest/developerguide/>
- Terraform AWS provider (`aws_cognito_user_pool`, `aws_cognito_user_pool_client`) — <https://registry.terraform.io/providers/hashicorp/aws/latest/docs>
- Cognito user pools API reference — <https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/>

Bump the `version` in `.claude-plugin/plugin.json` with any change to `templates/` or `skills/`: the platform pins a pack by version and content hash, and gates each version's templates once.

## Licensing

All content in this plugin is original Albitor material licensed under MIT — see [`LICENCE`](LICENCE).
