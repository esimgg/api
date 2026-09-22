# esim.gg Customer API

Base URL:

```text
https://api.esim.gg/api
```

## Authentication

Sign in to esim.gg, then open **https://esim.gg/settings/api-keys** directly to create, list, or revoke keys. The page is intentionally not linked from website navigation or settings; this documentation is where to find it.

Enter a name and create your key. Copy the secret immediately: it is shown only once. Send it as a bearer token:

```http
Authorization: Bearer <YOUR_API_KEY>
```

For line-scoped endpoints, also send the line number in `X-MSISDN`:

```http
X-MSISDN: 372XXXXXXXX
```

Use placeholders in examples; never put a key in source control or client-side code.

## Endpoint index

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/wallet/balance?currency=eur` | Read wallet balance |
| POST | `/number/search` | Search available numbers (rate limited) |
| POST | `/checkout/new_line` | Order a standard global number from the wallet |
| GET | `/line/all` | List your lines |
| GET | `/line/get_line` | Get the line selected by `X-MSISDN` |
| GET, POST | `/line/status` | Read or change service status |
| POST | `/line/nickname` | Set line nickname |
| POST | `/line/transfer_ownership` | Transfer a line to another account |
| POST | `/line/balance_transfer` | Transfer airtime credit between lines |

## Wallet balance

```bash
curl 'https://api.esim.gg/api/wallet/balance?currency=eur' \
  -H 'Authorization: Bearer <YOUR_API_KEY>'
```

Response:

```json
{"currency":"EUR","balance":12.34}
```

The balance is wallet credit. A number being zero-price does not provide free airtime or card credit.

## Search available numbers

Search is rate limited to **10 requests per minute per user**, across all of that user’s keys, in addition to the existing IP-based limit. By default, search includes both paid and zero-price numbers. Set the optional boolean `zero_price_only` field to `true` to return only numbers whose number price is zero.

```bash
curl -X POST 'https://api.esim.gg/api/number/search' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'Content-Type: application/json' \
  -d '{"search":"37255","type":"global","zero_price_only":true}'
```

Search results use this format (the number below is a placeholder):

```json
{"success":true,"search":[{"msisdn":"372XXXXXXXX","price":0.0}]}
```

`search` is an optional digit pattern; `type` is `global` or `asia`; and `zero_price_only` is an optional boolean that defaults to `false`. At most 12 results are returned. When `zero_price_only` is `true`, the zero-price filter is applied before the result limit and the search bypasses cached results. A zero number price does not waive activation, package, or airtime charges. A result is not a reservation: check the order response.

## Order a number

Orders use wallet credit only. This endpoint is for standard global numbers; it does not accept premium-number checkout or card/airtime payment flows.

```bash
curl -X POST 'https://api.esim.gg/api/checkout/new_line' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'Content-Type: application/json' \
  -d '{"msisdn":"372XXXXXXXX","payment_method":"wallet","recharge_amount":"2.00"}'
```

`recharge_amount` is required. The standard minimum is `1.00` EUR, although an account-specific minimum may apply. The `msisdn` must be a standard global number returned by search.

If an order does not return a final line result, do not blindly retry. First call `/line/all` and reconcile line ownership; a retry could create a second order.

## List and read lines

List the lines owned by the authenticated user:

```bash
curl 'https://api.esim.gg/api/line/all' \
  -H 'Authorization: Bearer <YOUR_API_KEY>'
```

The response is an object with a `lines` array (not a bare array):

```json
{"lines":[{"number":"372XXXXXXXX"}]}
```

The example shows selected fields only. Each line object includes `number` and may include service, nickname, and balance details. Use `number` as the `X-MSISDN` header value.

Read one line selected by `X-MSISDN`:

```bash
curl 'https://api.esim.gg/api/line/get_line' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX'
```

## Service status

Get current service status:

```bash
curl 'https://api.esim.gg/api/line/status' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX'
```

Update one service. `type` must be `voice`, `gprs`, `sms`, or `voicemail`; `value` must be boolean. GPRS cannot be disabled.

```bash
curl -X POST 'https://api.esim.gg/api/line/status' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX' \
  -H 'Content-Type: application/json' \
  -d '{"type":"gprs","value":true}'
```

## Nickname

Set the nickname for the selected line (at most 64 characters), or send `{"nickname":null}` to clear it:

```bash
curl -X POST 'https://api.esim.gg/api/line/nickname' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX' \
  -H 'Content-Type: application/json' \
  -d '{"nickname":"Travel line"}'
```

## Transfer line ownership

Ownership transfer is irreversible and has no recipient acceptance step. The recipient must already have an esim.gg account. The authenticated owner initiates the transfer.

Send exactly one of `recipient_email` or `recipient_account_id`. Email transfers succeed only when the email identifies exactly one account. If multiple accounts share that email, the transfer fails without changing ownership; use the recipient's account ID instead. Do not send both fields.

To find an account ID, the recipient signs in and long-presses their user/email area in the website header or sidebar. They can then copy the displayed account ID and share it with the sender. Verify it with the intended recipient before transferring.

Transfer by email:

```bash
curl -X POST 'https://api.esim.gg/api/line/transfer_ownership' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX' \
  -H 'Content-Type: application/json' \
  -d '{"recipient_email":"recipient@example.invalid"}'
```

Transfer by account ID:

```bash
curl -X POST 'https://api.esim.gg/api/line/transfer_ownership' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX' \
  -H 'Content-Type: application/json' \
  -d '{"recipient_account_id":"<RECIPIENT_ACCOUNT_ID>"}'
```

A successful response includes `success: true` and `transfer_details`, including the line number, previous/new owner identifiers, recipient email, transfer time, and line information.

## Transfer balance between lines

Transfer airtime credit from the selected line to another eligible line. This is line service balance, not the account wallet balance. The amount must be a positive EUR amount with two decimal places. The destination is supplied as `to` (an 8-digit local number or full number, as accepted by the service).

```bash
curl -X POST 'https://api.esim.gg/api/line/balance_transfer' \
  -H 'Authorization: Bearer <YOUR_API_KEY>' \
  -H 'X-MSISDN: 372XXXXXXXX' \
  -H 'Content-Type: application/json' \
  -d '{"to":"372YYYYYYYY","amount":"1.00"}'
```

Success returns `{"success":true,"message":"Transfer successful"}`. Transfers can be rejected when the destination is unavailable or the lines are not eligible together.

## Errors and security

Errors are JSON objects with an `error` code and sometimes a `message`; validation and permission failures commonly use HTTP 400, 403, or 404, while conflicts/rate limits may use 409 or 429. Treat non-2xx responses as failures and avoid automatic purchase retries.

Use HTTPS, keep keys private, grant only the access needed, and revoke a key immediately if exposed. The API-key allowlist is limited to the endpoints in this document; dashboard-only features are not implied to be available through API keys.
