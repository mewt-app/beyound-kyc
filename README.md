# beyound-kyc

Static landing for KYC verification. Served at https://kyc.beyound.in
(GitHub Pages + CNAME).

## How it works

1. User opens `kyc.beyound.in`, clicks **Start KYC verification**.
2. Page calls `POST /kyc/token` on the backend (see `CONFIG.tokenEndpoint`
   in `index.html`). Default URL: `https://api.beyound.in/kyc/token`.
3. Backend mints a short-lived HyperVerge JWT and returns
   `{ token, workflowId?, transactionId? }`.
4. Page boots `HyperKYCModule.launch(cfg, callback)`. SDK runs the
   verification flow inline. Result is surfaced in the status panel.
5. Authoritative decisioning happens on the backend via the HyperVerge
   **Results Webhook** — not from this page (avoid MITM, per HV docs).

## Files

| File         | Purpose                                                        |
|--------------|----------------------------------------------------------------|
| `index.html` | Single-page launcher. Loads HyperVerge Web SDK from CDN.       |
| `CNAME`      | GitHub Pages custom domain (`kyc.beyound.in`).                 |

## Deployment

GitHub Pages: serve from `main` branch root. The `CNAME` file pins
`kyc.beyound.in`.

Required DNS (verified against `mewt-app/beyound`):

```
kyc.beyound.in.   CNAME   mewt-app.github.io.
```

GitHub Pages issues the Let's Encrypt cert automatically once the CNAME
resolves. After that the API lets you flip `https_enforced: true`:

```
gh api -X PUT repos/mewt-app/beyound-kyc/pages -f https_enforced=true
```

## Backend integration TODO

The page calls `https://api.beyound.in/kyc/token` by default. Wire up a
backend route that:

1. Validates the requesting session (logged-in beyound user only).
2. Calls HyperVerge's `POST https://auth.hyperverge.co/login` (or the
   newer JWT-mint endpoint) with the workspace `accessKey` + `appId`
   stored as backend secrets.
3. Returns `{ token, workflowId, transactionId }` to the browser.

Workspace credentials and webhook URL are configured in the HyperVerge
One dashboard.

## Overriding the token endpoint

To point the page at a staging backend without forking, set the
endpoint URL on `window` BEFORE the inline script runs:

```html
<script>window.BEYOUND_KYC_TOKEN_ENDPOINT = "https://staging.api.beyound.in/kyc/token";</script>
```

## Calling app (`app` query param / header)

Apps that open this page pass their context on the URL:

```
https://kyc.beyound.in/?sid=<session-id>&mid=<merchant-id>&app=<spaid|rally>&return_url=<url>
```

`app` names the calling product. The page forwards it as an HTTP header
named `app` on every su-api-v2 call it makes
(`POST /api/v1/zokudo/wallet/register` and
`POST /api/v1/zokudo/wallet/generate-kyc-request`, including the automatic
retry):

```
app: spaid
```

- The value is trimmed and lower-cased (`?app=RALLY` is sent as `app: rally`),
  then accepted only if it matches `^[a-z0-9_-]{1,32}$`.
- A missing or invalid value counts as absent: the `app` header is omitted
  entirely (never sent as an empty string), an invalid value is logged once
  with `console.warn`, and the other headers and the request body are unchanged.
- The page keeps no list of valid app names; the backend decides which values
  it recognises.
- `app` is a custom request header, so the browser preflights it. The API's
  `Access-Control-Allow-Headers` must include `app` for origin
  `https://kyc.beyound.in`, otherwise the browser blocks both calls whenever a
  caller passes `?app=`.

## SDK version

Pinned to `@latest` from `https://hv-web-sdk-cdn.hyperverge.co/`. For a
specific version, replace `@latest` in `index.html` with a SemVer from
the [Web SDK Changelog](https://documentation.hyperverge.co/identity-verification-platform/sdk-integration/web-sdk/changelog).
