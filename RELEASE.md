# Release Notes

## 0.1.3 (unreleased)

- Hardening (BOT-016 reviewer note, not exploitable since the value is
  SQS's own system-set numeric attribute, not attacker input): `lease`'s
  `attempts` computation interpolated `${receiveCount}` directly into a
  `node -e` shell string. Removed the last raw-`${}`-in-shell spot by
  passing the value as a real `node` argv (`value(receiveCount)` ->
  `process.argv[1]`) instead. Same 0-indexed `attempts` output as before.

## 0.1.2 (unreleased)

- Fixed a real bug: `lease`'s `attempts` field reported SQS's raw (1-indexed)
  `ApproximateReceiveCount` directly, while `aux4/queue`'s memory and
  repository providers report a 0-indexed "attempts already made" count.
  `aux4/bot`'s shared `isFinalAttempt` formula (`attempts + 1 >= maxAttempts`)
  assumes the 0-indexed convention, so on `sqs` it fired one delivery early
  and then AGAIN on the next delivery before SQS's own redrive policy moved
  the message to its dead-letter queue — a `failureMessage` could be sent
  twice for the same event. `lease` now reports `attempts =
  ApproximateReceiveCount - 1`, matching every other provider.

## 0.1.0 (unreleased)

- Renamed the dispatch flag: `aux4/queue` now selects this plugin with `--provider sqs`; the
  original `--storage sqs` spelling still works as a backward-compatible alias (see `aux4/queue`'s
  own `--provider`/`--storage` change).
- Removed `printf`/shell-format-string JSON construction from `lease` and `dlq` — `lease` now emits
  its JSON line via the `log:` executor (never spawns a shell, so untrusted message bodies can't
  reach a shell parser), and `dlq` dispatches a private per-item `dlq-item` command (receiving each
  field as a real subprocess argument) instead of interpolating AWS response fields into a format
  string.
- `lease` now exits `4` (not `1`) when nothing is eligible, matching `aux4/queue`'s own
  memory/repository convention.

## 0.1.0 (original draft)

- Initial release: Amazon SQS provider plugin for `aux4/queue`'s work-queue mode.
- Adds `--storage sqs` to `aux4 queue create/delete/list/send/lease/ack/nack/renew/dlq`, backed by
  real Amazon SQS FIFO queues (plus an automatic `<name>-dlq.fifo` dead-letter queue wired via an
  SQS redrive policy).
- Credentials, region, and endpoint are resolved entirely through the AWS CLI's own standard
  mechanism (env vars, profile, or role); `AWS_ENDPOINT_URL` is supported for local testing against
  an SQS-compatible server (ElasticMQ, LocalStack).
- `renew` extends a lease's visibility timeout without acknowledging or releasing the message.
