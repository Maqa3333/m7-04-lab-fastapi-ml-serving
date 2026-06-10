# API Design Decisions

## 1. Versioning — Path-based (`/v1/`) over header-based

Path-based versioning (`/v1/predict`) is visible in every log line, proxy rule, and browser request without any special tooling — a developer can `curl` the right version without remembering a header name. Header-based versioning (e.g. `Accept: application/vnd.visionmod.v1+json`) is more RESTfully "correct" but creates friction for partner integrations and makes routing rules in API gateways and load balancers meaningfully harder to write.

## 2. Batch ordering and partial failures

When `predict-batch` receives 32 items and one image is corrupt, the API returns HTTP **200** with a mixed `results` array: every successfully classified item includes a `labels` array, and the corrupt item includes an `error` object (code `INVALID_IMAGE`) in place of labels. The response always contains exactly one entry per submitted item, keyed by the caller-supplied `id`, so callers never have to guess which result maps to which input. This "continue on error" policy was chosen over fail-fast because batch callers are typically processing user-upload queues where a single bad file should not block the 31 valid ones; clients inspect each result's `error` field to detect and re-queue failures.

## 3. Async lifecycle

A job transitions through four states: `queued` (accepted, waiting for a worker) → `processing` (inference in progress) → `completed` (labels available) or `failed` (unrecoverable error). Results are retained for **24 hours** after reaching a terminal state (`completed` or `failed`), after which `GET /v1/predictions/{job_id}` returns 404; this window is long enough for any reasonable retry loop or webhook failure scenario without creating unbounded storage growth. Jobs that remain stuck in `queued` or `processing` for more than **10 minutes** are automatically transitioned to `failed` with code `SERVICE_UNAVAILABLE`, preventing silent hangs.
