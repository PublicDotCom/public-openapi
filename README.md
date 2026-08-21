# Public.com API — OpenAPI Specification

The official OpenAPI 3.0.1 description of the [Public.com](https://public.com) REST API. This
repository is the source of truth for the spec: use it to explore the API, generate client
libraries, or import the endpoints into your HTTP client of choice.

- **Spec:** [`spec.yaml`](spec.yaml)
- **Base URL:** `https://api.public.com`
- **Version:** `1`

## What's covered

| Area | Endpoints |
| --- | --- |
| Authorization | Exchange a personal secret for a short-lived access token |
| Accounts | List accounts; retrieve a portfolio snapshot and account history |
| Instruments | Look up equities, options, and other instruments; filtered fixed-income search |
| Market data | Real-time quotes, option chains and expirations, bond details, historical bars |
| Options | Greeks and multi-leg strategy quotes |
| Orders | Preflight, place, replace, cancel, and retrieve single- and multi-leg orders |
| Tax lots | Unrealized tax lots by account or symbol, including CSV export |

## Authentication

Every endpoint except token creation requires a bearer token.

1. **Generate a personal secret** from your Public.com account settings. Secrets are long-lived
   but revocable — treat one like a password and never commit it.
2. **Exchange the secret for an access token** via
   `POST /userapiauthservice/personal/access-tokens`.
3. **Send the token** as `Authorization: Bearer <accessToken>` on every subsequent request.

Access tokens are short-lived: validity is configurable between **5 and 1440 minutes** and
defaults to **15 minutes**. Build refresh into your client rather than caching a token
indefinitely — expired tokens return `401`.

```bash
# 1. Get an access token
ACCESS_TOKEN=$(curl -s -X POST https://api.public.com/userapiauthservice/personal/access-tokens \
  -H 'Content-Type: application/json' \
  -d '{"secret":"YOUR_PERSONAL_SECRET","validityInMinutes":60}' \
  | jq -r '.accessToken')

# 2. Find your accountId — most endpoints are scoped to one
ACCOUNT_ID=$(curl -s https://api.public.com/userapigateway/trading/account \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  | jq -r '.accounts[0].accountId')

# 3. Call an endpoint
curl -s -X POST "https://api.public.com/userapigateway/marketdata/$ACCOUNT_ID/quotes" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"instruments":[{"symbol":"AAPL","type":"EQUITY"}]}'
```

The `accountId` returned by `GET /userapigateway/trading/account` is stable for the lifetime of
the account, so you can resolve it once and store it.

## Using the spec

**Browse the docs locally** with any OpenAPI renderer:

```bash
npx @redocly/cli preview-docs spec.yaml
```

**Generate a client** in your language of choice:

```bash
npx @openapitools/openapi-generator-cli generate \
  -i spec.yaml -g typescript-fetch -o ./client
```

**Import into Postman or Insomnia** by pointing the importer at `spec.yaml` (or the raw GitHub
URL) — both read OpenAPI 3.0 natively.

**Validate** before relying on a local edit:

```bash
npx @redocly/cli lint spec.yaml
```

## Working with orders

Order-placing endpoints take a client-supplied `requestId` — a UUID (RFC 4122) that must be
globally unique over time. Reusing the same `requestId` on the same account makes the operation
idempotent, which is what makes it safe to retry a request that failed with a read timeout.
When retrying, resend the request **unchanged**; altering fields has no effect if the original
call already succeeded.

The `preflight/single-leg` and `preflight/multi-leg` endpoints estimate the financial impact of
a trade before you commit to it. Running preflight first is recommended for any order flow that
surfaces cost or buying-power impact to a user.

## Errors

Errors return a JSON body with a machine-readable `errorCode` and a human-readable `message`:

```json
{
  "errorCode": "user_api_auth.personal.invalid_secret",
  "message": "Invalid secret"
}
```

Branch on `errorCode`, not on `message` — messages may be reworded without notice. Token
creation is rate limited and returns `429` when you exceed it; back off before retrying.

## Versioning

The spec is versioned via `info.version` in [`spec.yaml`](spec.yaml). Changes are published to
this repository, so watch it or subscribe to releases to be notified when the API changes.

## Feedback

Found an inaccuracy in the spec, or something that doesn't match the API's actual behavior?
Open an issue in this repository. For account-specific questions or anything involving your
credentials, contact Public.com support directly rather than filing a public issue.
