# Envelope webhooks demo

Playwright `recordVideo` of a local SignuDoc install (Postgres 17, `EMAIL_DRIVER=capture`, receiver at `http://127.0.0.1:43721/hooks`). The operator is Ada Admin. The receiver verifies `SignuDoc-Signature` with the secret shown on create.

`out.mp4` is 1920×1080, 30 fps, H.264, yuv420p, 35.80 s (1074 frames). `preview.gif` is 960×540, 6 fps, 2 320 448 bytes.

## What the recording shows

1. Settings → Webhooks: add `http://127.0.0.1:43721/hooks` as “Local receiver”. The one-time signing secret is shown.
2. Send test event. The receiver lists `envelope.completed` with `"test": true` and `signature accepted`.
3. Envelope “Office lease” is uploaded, sent to Riley Recipient, and signed. The receiver lists `envelope.completed` for that title, signature accepted.
4. Envelope “Board resolution” is voided with reason “No longer needed.” The receiver lists `envelope.voided`, signature accepted.
5. The receiver is switched to HTTP 500. Send test event leaves a deliveries row: Completed, Pending, HTTP 500, 1 attempt, “The endpoint responded with 500.”
6. Adding `http://10.0.0.5/hooks` is refused with “That address cannot be used as a webhook URL.”

Stills from the same run are in `screenshots/`. `07-final-deliveries.png` is the refused URL (the form scrolled that banner into view at the end). `commands.log` and `logs/` are the check outputs, including `logs/ffprobe.txt`.

## Not verified in this recording

Delivery from a Vercel deployment, the Hobby daily cron firing in production, and a POST to a public HTTPS hostname.
