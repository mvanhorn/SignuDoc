Copy a finished envelope into a new draft

## What this adds

A sender can start a new draft from an envelope that has already left draft: sent, completed, declined, voided, or past its deadline. **Copy to new draft** on the envelope page (and `POST /api/envelopes/:id/copy`) makes a new draft for the same owner, then opens the usual prepare/review flow. Nobody is emailed. The original envelope is left as it is.

The new draft keeps:

- the same title
- the original unsigned PDF, copied to a new storage key (same bytes, different file)
- the same signers in the same order, each pending, with no signing tokens
- the same copy recipients
- the emailed-code check on each signer
- every field (type, page, position, size, required, choices, assignment), with no filled-in values

Access codes are not copied. They are stored hashed, so they cannot be brought across. The confirmation dialog says so, and new codes can be set before sending.

Deadline, reminders, and Part 11 are not copied. Those are chosen again when the new draft is sent, the same as any other draft. The thumbnail is not shared: it is generated again for the new envelope.

The copy is recorded as a `copied` event on the new draft only. The original audit chain is not rewritten. A new migration re-adds `audit_events_event_check` with the full event list plus `copied`; released migrations are not edited.

## Edge behavior

- Only the owner can copy. Anyone else, including another signed-in sender, gets 404.
- A draft cannot be copied (409). The button is hidden on drafts.
- If the original PDF is gone, the copy is refused (409) and no half-made draft is left. The file is written first; a failed transaction deletes that copy.
- Deleting either envelope does not remove the other's PDF.
- Copying does not send mail. In the live run the capture outbox was unchanged after `POST /copy`.

## Tests

- `npm test`: 214 pass, 0 fail.
- `npm run test:db`: 110 pass, 0 fail. New `src/lib/envelope-copy.dbtest.ts` (6 tests) covers a completed copy (people, fields without values, separate storage key, equal PDF hash, original rows unchanged, deleting one file leaving the other), declined/voided/sent/past-deadline copies, draft refusal, non-owner 404, a missing original PDF with no leftover draft, and Zod rejection of a blank id.
- `npm run typecheck`: clean.
- `npm run lint`: 0 errors, 5 warnings, all in `prepare-live-shell.tsx` on the base branch.
- `npm run format:check`: 14 files already fail on `main` (including the vendored pdf.js worker). None of the files in this change are in that list.
- `npm run build`: succeeds. The route list includes `/api/envelopes/[id]/copy`.

A live run against local Postgres 17, `EMAIL_DRIVER=capture`, and disk storage created a completed envelope with two signers, one copy recipient and fields, then `POST /api/envelopes/<id>/copy` returned `{"id":"db699bf5-6a9b-4196-81d6-17a059390948"}`. The new draft was status `draft` with the same title; signers were pending with no tokens; field values were null; storage keys differed; SHA-256 of both original PDFs was `89301ab0c3872dc8a9537f0fc266e42adb5a9f5d94a819115ca97eeb2c889c41`. Copying a leftover draft returned 409. Copying as a second verified sender returned 404. Commands, JSON dumps, Playwright screenshots and a HyperFrames replay of that captured output are on `demo/copy-envelope-as-draft`.

## Not checked

- A Vercel (or other hosted) deployment, Blob storage, or Resend. The live run used local disk and the capture outbox.
- `npx playwright test` / `npm run test:e2e`. Screenshots of the copy button, the confirmation dialog, and the opened draft were taken against `next dev` instead.
- Copying an envelope that had an access code, in the live HTTP run. The dialog states they are not copied; the hashed value is not readable to copy.
- `requireEmailCode: true` in the live HTTP run. The flag is copied; the database tests cover it. The completed demo envelope had it off so the signers could finish without an emailed code.
- Repo-wide `format:check` was not made green. The 14 failing files were already failing on `main` and were left alone.
