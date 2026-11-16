Title: Send signed webhooks when an envelope is completed, declined, voided or expires

An install can tell its own systems when a document is finished. Until now the only outbound signal was email. This adds administrator-configured HTTPS webhooks, signed with a secret shown once, for completed, declined, voided and expired envelopes.

## What changed

- Settings grows a Webhooks panel for administrators: add one or more endpoints, pick events, turn an endpoint on or off, rotate the signing secret, send a test event, and retry a failed delivery.
- Completing, declining, voiding or expiring an envelope writes a delivery row in the same transaction as the status change and POSTs after commit.
- Failed deliveries are retried from the existing daily cleanup at `/api/internal/retention`. There is no new cron entry.
- Migration `20261117090000_webhooks.sql` adds `webhook_endpoints` and `webhook_deliveries`. Endpoint changes in Part 11 mode go to `system_audit_events`.
- The self-hosting guide documents the payload, headers, signature check and URL rules. CHANGELOG has an Unreleased entry.

## How it behaves

**Events.** `envelope.completed`, `envelope.declined`, `envelope.voided`, `envelope.expired`. A disabled endpoint, or an event that endpoint did not subscribe to, sends nothing.

**Payload.** JSON with event id, type, occurred-at, envelope id, title, status, sender email, signer count, and a link to the envelope page. A test POST also includes `"test": true`. Document bytes, signer tokens and access codes are not included.

**Signature.** Headers `SignuDoc-Event`, `SignuDoc-Delivery`, `SignuDoc-Timestamp` and `SignuDoc-Signature: v1=<hex>`. The hex is HMAC-SHA256 of `"<timestamp>.<raw body>"` with the current secret. Verify it against the raw body, not a re-serialized object. The POST follows no redirects and waits eight seconds. A 2xx response counts as delivered.

**Retries.** A non-2xx response, a timeout or a network error leaves the row pending. The daily job waits 1, 2, 4, then 8 days, up to five attempts, then marks the delivery failed. Send again posts the stored body immediately.

**Refusals.** URLs must be HTTPS. `http://localhost`, `http://127.0.0.1` and `http://[::1]` are allowed only outside production. Loopback, private, link-local and metadata hosts are refused, including `http://10.0.0.5`.

## Testing

Recorded on the untouched upstream tip `8d607a6` first, then on `feat/envelope-webhooks`.

```
npm run typecheck
npm run lint
npm run format:check
npm test
npm run test:db
npm run build
```

- `npm run typecheck`: pass on upstream, pass on this change.
- `npm run lint`: 0 errors, 5 warnings on upstream; 0 errors, 5 warnings on this change (same files, `prepare-live-shell.tsx`).
- `npm run format:check`: 14 dirty files on upstream, all already on main. The same 14 remain. The files this change adds or edits, other than the SQL migration (Prettier has no parser for it), pass `prettier --check`.
- `npm test`: 281 pass on upstream, 292 pass on this change (+11 in `src/lib/webhooks.test.ts`).
- `npm run test:db`: 114 pass on upstream, 122 pass on this change (+8 in `src/lib/webhooks.dbtest.ts`). Database tests use a local HTTP receiver.
- `npm run build`: pass on this change, including `/api/admin/webhooks`, `/api/admin/webhooks/[id]`, rotate, test, and deliveries retry.

## Not verified

- Delivery from a Vercel deployment, including the Hobby daily cron actually firing in production.
- Resolving a public HTTPS hostname and posting to it from a production install.
- A receiver rejecting old `SignuDoc-Timestamp` values (the self-hosting guide tells you to check the clock; the app does not).

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/envelope-webhooks/preview.gif)

Full recording (MP4): https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/envelope-webhooks/out.mp4
