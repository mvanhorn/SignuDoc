Title: Send one template to many signers from a CSV

A sender who already has a template still had to use it once per person. Send to many takes a CSV of names and addresses, shows which rows will go out, and sends one envelope per valid row from that template.

## What changed

- **Send to many** on each template: upload or paste a CSV, check the list, then send.
- One envelope per valid row, created from the template and sent on the same path as a hand-sent draft (consent, audit trail, deadline, reminders, copy recipients, email code).
- A results table with sent / skipped / failed, links to the envelopes, and a downloadable results CSV.
- No migration. Part 11 mode skips a row whose signer cannot sign, and says why.

## How it behaves

- One-signer templates accept `name,email` (optional `title`). Multi-role templates need `<role> name` and `<role> email` for each role.
- Quoted commas, doubled quotes, CRLF and a UTF-8 BOM are read as written. A batch can send at most 200 envelopes; extra rows are skipped.
- Rows are checked before anything is created. Missing or invalid addresses, duplicate rows, and Part 11 failures are skipped with a reason.
- Confirming sends rows one after another. A failure on one row does not take the others with it. Already-created drafts of a failed row are kept.
- Preview is limited to 30 checks per 10 minutes; sending a batch is limited to 10 per hour.

## Testing

Baseline on untouched `upstream/main` (`8d607a671b055c44a1d4f8241072d1e2261fc8bc`):

- `npm run typecheck` — pass
- `npm run lint` — 0 errors, 5 warnings (`prepare-live-shell.tsx`, pre-existing)
- `npm run format:check` — 14 files already dirty
- `npm test` — 281 pass
- `npm run test:db` — 114 pass

On this branch:

- `npm run typecheck` — pass
- `npm run lint` — 0 errors, 5 warnings (the same five)
- `npx prettier --check` on the changed files — pass; `npm run format:check` still reports the original 14
- `npm test` — 289 pass
- `npm run test:db` — 120 pass
- `npm run build` — pass

CSV parser tests cover quoted commas, CRLF, a UTF-8 BOM, one- and multi-role headers, skip reasons, the 200-row cap, and formula-safe results. DB tests cover a 3-row send (fields, deadline, reminders, CC, email code), a bad row skipped while the rest send, the cap with nothing created, another user's template refused, Part 11 skip, and sending without the permitted-use checkbox.

## Not verified

- A 200-row send in the browser (the cap is covered in unit and DB tests)
- Part 11 and multi-role CSV in the browser (covered in DB tests)
- Real Resend delivery (`EMAIL_DRIVER=capture` locally)
- Advanced or qualified signatures on a bulk send
- The 10-batch-per-hour limit in the UI

## Demo

![demo](https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/bulk-send-template/preview.gif)

Full recording (MP4): https://raw.githubusercontent.com/mvanhorn/pr-demo-assets/main/osc-bulk-1116/bulk-send-template/out.mp4
