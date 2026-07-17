Distilled reference for the `cognito-auth` skill. Verify current Cognito SDK/API names against the AWS SDK for JavaScript v3 (`@aws-sdk/client-cognito-identity-provider`) docs before shipping.

# In-app auth UI: sign-up, login, confirm-code, resend

The screens live **in the app** and are styled by the app's chosen design-system skill — **never the Cognito Hosted UI**. This skill owns the flow and the calls; the design-system skill owns the look. Reach for that skill's form field, button, and error-summary components for every control below.

Build the screens in this order — the done-criterion is that a **first-time visitor can self-register**, so sign-up comes first.

## Talking to Cognito from the browser

Use the AWS SDK v3 `@aws-sdk/client-cognito-identity-provider` (unauthenticated commands — no AWS credentials needed for these calls). The only config the client needs is the **region** and the public **app client id** from the Terraform outputs.

```ts
import {
  CognitoIdentityProviderClient,
  SignUpCommand,
  InitiateAuthCommand,
  ConfirmSignUpCommand,
  ResendConfirmationCodeCommand,
} from "@aws-sdk/client-cognito-identity-provider";

const cognito = new CognitoIdentityProviderClient({ region: CONFIG.cognitoRegion });
const CLIENT_ID = CONFIG.cognitoAppClientId; // public, safe in the browser
```

(You can equally use Amplify Auth; the flows and screens are identical. Keep whichever the app's stack already leans on.)

## 1. Sign-up screen (build this first)

Form: email + password (+ confirm-password). On submit, call `SignUpCommand`.

```ts
await cognito.send(new SignUpCommand({
  ClientId: CLIENT_ID,
  Username: email,
  Password: password,
  UserAttributes: [{ Name: "email", Value: email }],
}));
```

- **Default (email verification required):** the new user is created but unconfirmed, and Cognito emails a confirmation code. Route straight to the **confirm-code screen** (§3) — this is the normal happy path.
- **Dev/CI auto-confirm override only:** when the app's dev environment runs with the documented auto-confirm override (see `references/terraform.md`), the user is immediately usable — redirect to login (or auto-login) and skip the code screen. This is a development shortcut, not the default.
- Surface Cognito errors (`UsernameExistsException`, `InvalidPasswordException`, `InvalidParameterException`) through the design system's error-summary pattern with human-readable messages.

## 2. Login screen

Form: email + password. Use the `USER_PASSWORD_AUTH` flow.

```ts
const out = await cognito.send(new InitiateAuthCommand({
  ClientId: CLIENT_ID,
  AuthFlow: "USER_PASSWORD_AUTH",
  AuthParameters: { USERNAME: email, PASSWORD: password },
}));
const { AccessToken, IdToken, RefreshToken } = out.AuthenticationResult!;
```

Store the tokens (see below), then send the user to the app's authenticated area. Handle `NotAuthorizedException` (bad credentials) and `UserNotConfirmedException` — under the verification-required default this is a normal case (the user signed up but hasn't entered their code yet), so route them to the confirm-code screen (§3).

## 3. Confirm-code screen (default happy path)

Form: the 6-digit code. Call `ConfirmSignUpCommand`.

```ts
await cognito.send(new ConfirmSignUpCommand({
  ClientId: CLIENT_ID,
  Username: email,
  ConfirmationCode: code,
}));
```

This screen is on the **default happy path**: after sign-up the user lands here, enters the emailed code, and is confirmed before they can log in. It is only skipped when the dev/CI auto-confirm override is active (see `references/terraform.md`), so build it as a first-class screen, not an afterthought.

## 4. Resend-code screen / action

A "didn't get a code?" action that calls `ResendConfirmationCodeCommand`.

```ts
await cognito.send(new ResendConfirmationCodeCommand({
  ClientId: CLIENT_ID,
  Username: email,
}));
```

## Storing and using tokens

**The session must survive a page reload, and deep-links must work.** A user who
refreshes the tab or opens a bookmarked protected URL stays signed in — this is a
non-negotiable baseline, not a preference. Losing the session on reload is a bug.

- Keep the **access token** for API calls and send it as `Authorization: Bearer <accessToken>` (the API middleware verifies it — see `references/api.md`).
- **Baseline (required): persist the refresh token in `localStorage` and rehydrate on load.** The access/id tokens may live in memory, but the **refresh token** is written to `localStorage` so that on app start you can silently re-mint fresh access/id tokens via `REFRESH_TOKEN_AUTH` (below) before rendering the authenticated area. This is what makes reload and deep-links work, and it matches Amplify Auth's default persistence — so if the app uses Amplify, its default already satisfies this baseline.
- **On load:** read the stored refresh token; if present, run a `REFRESH_TOKEN_AUTH` call to rehydrate the in-memory access/id tokens, then treat the user as signed in. If it fails (expired/revoked), clear the stored token and route to login.
- **Hardening for higher-security deployments (recommended): HttpOnly refresh-cookie + backend token endpoint (BFF).** Keep the refresh token out of JavaScript entirely by having a small backend set it as an `HttpOnly`, `Secure`, `SameSite` cookie and exposing a token endpoint the SPA calls to mint access tokens. The XSS tradeoff is plain: a refresh token in `localStorage` is readable by any injected script, whereas an `HttpOnly` cookie is not — so security-sensitive deployments should prefer the BFF pattern. But **survive-reload is the baseline either way**; the BFF is how you harden it, not an excuse to drop persistence back to memory-only.
- Refresh with `InitiateAuthCommand` + `AuthFlow: "REFRESH_TOKEN_AUTH"` before expiry (access/id tokens last 60 min by default), and on app load as described above.
- On logout, discard the in-memory tokens **and** remove the persisted refresh token (or clear the BFF cookie).

## Styling: consume the design-system skill

Every field, button, inline error, and error summary on these four screens comes from the app's selected design-system skill. Do not hand-roll form CSS and do not fall back to the Cognito Hosted UI. If no design-system skill is loaded when you build these screens, stop and ask which design system applies.
