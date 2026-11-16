# Demo: send one template to many signers from a CSV

Recorded against a local SignuDoc install (`EMAIL_DRIVER=capture`, Postgres 16, `next dev` on `http://127.0.0.1:3000`).

`out.mp4` is a Playwright Chromium recording at 1920×1080, 30 fps, H.264 yuv420p. `preview.gif` is the same recording scaled to 1280×720 at 10 fps.

## What the recording shows

1. The Templates list, with **Send to many** on Company NDA.
2. A four-row CSV pasted on Send to many (`name,email,title`). Row 2 uses `not-an-email`.
3. **Check this list**: three rows ready, Bob skipped with “Enter a valid email address.”
4. Confirm, including the ordinary-business-agreements checkbox, then **Send now**.
5. Results: 3 sent, 1 skipped, with links to the created envelopes.
6. Documents: the three sent envelopes (Ann, Cara, Dana).
7. The three signing invitations written to `.data/outbox`.

## Files

- `out.mp4` / `preview.gif` — the recording
- `screenshots/` — stills of each step
- `logs/` — typecheck, lint, format:check, unit tests, DB tests, build, ffprobe
- `commands.log` — the commands that produced those logs, with exit codes
- `PR_bulk-send-template.md` — suggested pull-request body (not opened)
