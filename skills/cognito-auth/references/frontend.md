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

- **Under auto-confirm (default):** the user is immediately usable. Redirect straight to login (or auto-login) — do **not** force a code screen.
- **Under real email verification (opt-in):** route to the confirm-code screen.
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

Store the tokens (see below), then send the user to the app's authenticated area. Handle `NotAuthorizedException` (bad credentials) and `UserNotConfirmedException` (only reachable under real email verification → send to confirm-code).

## 3. Confirm-code screen (built even under auto-confirm)

Form: the 6-digit code. Call `ConfirmSignUpCommand`.

```ts
await cognito.send(new ConfirmSignUpCommand({
  ClientId: CLIENT_ID,
  Username: email,
  ConfirmationCode: code,
}));
```

Under the auto-confirm default this screen is unreachable in the normal flow — but ship it, because enabling real email verification is a config flip (see `references/terraform.md`) that makes it live with no rework.

## 4. Resend-code screen / action

A "didn't get a code?" action that calls `ResendConfirmationCodeCommand`.

```ts
await cognito.send(new ResendConfirmationCodeCommand({
  ClientId: CLIENT_ID,
  Username: email,
}));
```

## Storing and using tokens

- Keep the **access token** for API calls and send it as `Authorization: Bearer <accessToken>` (the API middleware verifies it — see `references/api.md`).
- Prefer in-memory storage with a refresh-on-load using the refresh token; if you must persist, weigh the XSS trade-offs of `localStorage`. Do not put tokens in non-`HttpOnly` cookies naively.
- Refresh with `InitiateAuthCommand` + `AuthFlow: "REFRESH_TOKEN_AUTH"` before expiry (access/id tokens last 60 min by default).
- On logout, discard the stored tokens client-side.

## Styling: consume the design-system skill

Every field, button, inline error, and error summary on these four screens comes from the app's selected design-system skill. Do not hand-roll form CSS and do not fall back to the Cognito Hosted UI. If no design-system skill is loaded when you build these screens, stop and ask which design system applies.
