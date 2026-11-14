Let a sender correct a signer's email after sending

## What this adds

On an envelope that is already out for signature, the owner can correct the name and/or email of a signer who has not signed or declined. The signing link already issued to that person stops working. If it is that signer's turn, a new invitation goes to the corrected address. The audit trail records the old and new values, in the hash chain, and the same sentence is used on the envelope timeline, the CSV export, and the certificate.

Drafts still use the existing signer list editor. Copy recipients are not changed here.

The envelope page shows **Change** on each waiting or pending signer of a sent envelope that is not a Part 11 record. The dialog posts `PATCH /api/envelopes/:id/signers/:signerId`, shows the API error if the save is refused, and reloads the page when it succeeds.

## Edge behavior

- Only the owner can do this. Another account is answered 404, the same as a missing envelope.
- The envelope must be `sent`, including past its deadline. Draft, completed, declined, and voided envelopes are refused with 409: only an envelope that is out for signature can have a signer corrected.
- The signer must be `pending` or `notified`. Signed and declined are refused. A copy recipient is 404.
- Name and email are Zod-checked. An address already used by another signer or copy recipient on the same envelope is refused. Sending the same name and the same address (case-insensitive) is a no-op: the stored casing stays, the link is not rotated, nothing is audited, and no email is sent.
- A Part 11 envelope is refused. The signer's identity is bound to their account.
- The opaque signing token is rotated (or cleared when the signer is still pending). The old link gets the normal invalid-link response (`This link has expired.` / `expired`). Consent and email-code state for that signer are cleared.
- If the signer is `notified`, a new invitation is sent now and `email_sent` is recorded. If email delivery fails, the signer row, token, and audit event are rolled back. If the signer is `pending`, no invitation is sent yet.
- New audit event `signer_corrected` with previous and new name and email, the signer id and order, and the acting sender. `verifyAuditChain` still passes.

Migration `db/migrations/20261116090000_signer_corrected.sql` drops and re-adds `audit_events_event_check` with the previous list plus `signer_corrected`. Released migrations are not edited.

## Tests

- `npm test`: 216 pass, 0 fail. New `src/lib/signer-correction.test.ts` covers the Zod input and the plain-words sentences.
- `npm run test:db` against Postgres 17 (`E2E_DATABASE_URL=postgres://signudoc:signudoc@127.0.0.1:5432/signudoc_e2e`): 116 pass, 0 fail. New `src/lib/signer-correction.dbtest.ts` covers a notified correction (token rotated, old link rejected through `getSigningPayloadForToken`, invitation captured), a pending correction with no invitation, the no-op, a past deadline, a duplicate email, signed/declined, a copy recipient, draft/completed/declined/voided, Part 11, a non-owner 404, schema refusal, email-failure rollback, and that `verifyAuditChain` passes.
- `npm run typecheck`: clean.
- `npm run lint`: 0 errors, the same 5 warnings as upstream `main`, all in the field editor.
- `npx prettier --check` of the files in this change: clean. `npm run format:check` on the whole tree still reports 14 files that were already that way on `upstream/main`.
- `npm run build`: succeeds. The production route table includes `PATCH /api/envelopes/[id]/signers/[signerId]`.

A local production build was also run against Postgres 17 with `EMAIL_DRIVER=capture` and disk storage: setup, draft, upload, send to a mistyped address, `PATCH` (200, `invitationSent: true`), `GET` of the old signing link (`410`, expired), a new captured invitation to the corrected address, the `Signer corrected` CSV/timeline line, then a signed-signer 409 and a Part 11 409. Playwright screenshots of the Change dialog and the timeline are on the `demo/correct-signer-email` branch.

## Not verified

- `npm run test:e2e` was not run. The live walk used curl against `next start` plus Playwright screenshots of the envelope page.
- The finished PDF certificate page was not opened in that live walk. The sentence is shared with the timeline and CSV, which were shown, and the certificate path is covered in code.
- Nothing was exercised on a Vercel deployment, Resend, or Blob storage.
- The Change dialog was screenshotted at desktop width only.
- Whole-tree `prettier --check` was not made green; the 14 reported files are unchanged from `upstream/main`.
