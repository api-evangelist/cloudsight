---
generated: '2026-08-09'
method: generated
name: Describe an image with CloudSight
description: Submit an image to CloudSight and poll until a natural-language caption, a skip reason, or a timeout comes back.
api: openapi/cloudsight-images-openapi.yml
operations: [postImages, getImage, repostImage]
source: >-
  Operation ids verified verbatim in openapi/cloudsight-images-openapi.yml,
  which is a faithful conversion of the API Blueprint CloudSight publishes at
  https://cloudsight.docs.apiary.io/api-description-document. Cross-cutting
  rules cite ../conventions/, ../errors/ and ../authentication/.
---

# Describe an image with CloudSight

CloudSight turns a picture into a sentence. The API is asynchronous: you submit an
image, get a token, and poll that token until the job reaches a terminal state.

## Auth

- Send `Authorization: CloudSight [key]` on every request. OAuth 1.0a is also
  supported, with the `image` parameter excluded from the signature. See
  `authentication/cloudsight-authentication.yml`.
- There is no test mode and no test key. Every call runs against production and
  consumes a credit — see `sandbox/cloudsight-sandbox.yml`.

## Steps

1. **Submit the image** — `postImages` (`POST /images`), base `https://api.cloudsight.ai/v1`.
   Choose exactly one delivery method:
   - `multipart/form-data` with an `image` file part (preferred);
   - `application/json` with `remote_image_url` (the URL must return a direct
     `200`; any `3xx` redirect is an error);
   - `application/json` with `image` as a base64 data URI (small images only).
   Optional context: `locale`, `language`, `device_id`, `latitude`, `longitude`,
   `altitude`, `ttl`, `focus_x`, `focus_y`. Focal points use North-West gravity —
   `(0,0)` is the upper-left corner — in relative (`0.0`–`1.0`) or absolute
   pixel coordinates.
   Keep images at or below 1024px with JPEG quality 5–8; larger images are
   resized server-side and slow the request.
   A `201` returns `token` and the stored image `url`, plus the
   `X-CloudSight-CreditBalance` and `X-CloudSight-Overage` headers.

2. **Wait, then poll** — `getImage` (`GET /images/{token}`). Sleep 5 seconds
   after submitting, then poll once per second while `status` is
   `"not completed"`.

3. **Branch on `status`, not on the HTTP code.** The poll returns `200` for
   every outcome:
   - `completed` — `name` holds the caption. Check `flags` for `adult`.
   - `skipped` — read `reason`: `offensive`, `blurry`, `dark`, `bright`,
     `unsure`, `close`.
   - `timeout` — the job expired.

4. **Repost only when it is warranted** — `repostImage`
   (`POST /images/{token}/repost`) re-queues a `timeout`, and is the documented
   best practice for a skip reason of `unsure` or `close`. Do it **once**.
   Do not repost `blurry`, `dark`, `bright` or `offensive` — those need a
   different image, not another attempt.

## Errors

- `422` carries a field-keyed envelope, e.g. `{"error": {"image": ["can't be blank"]}}`.
- No `401`, `429` or `5xx` behaviour is documented. See
  `errors/cloudsight-problem-types.yml`.

## Notes

- **No idempotency key exists.** Re-submitting the same image creates a new job
  and consumes another credit. Deduplicate on your side before calling
  `postImages`. See `conventions/cloudsight-conventions.yml`.
- **No webhooks.** Polling is the only completion signal.
- **Budget your polls.** `X-CloudSight-CreditBalance` and `X-CloudSight-Overage`
  on the submit response are the only usage signal the API returns.
