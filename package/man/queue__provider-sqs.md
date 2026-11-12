#### Description

`aux4/queue-sqs` is a provider plugin for `aux4/queue`. It adds `sqs` as a `--provider` choice to
every `aux4 queue` work-queue command — `create`, `delete`, `list`, `send`, `lease`, `ack`, `nack`,
`renew`, `dlq` — backing the queue with real Amazon SQS FIFO queues instead of `aux4/queue`'s
built-in `memory` or `repository` providers. There is no new top-level command: once this package is
installed alongside `aux4/queue`, pass `--provider sqs` to the commands you already use. The older
`--storage sqs` spelling still works as a backward-compatible alias.

Each queue created with `--provider sqs` is backed by two SQS FIFO queues:

- `<name>.fifo` — the queue itself.
- `<name>-dlq.fifo` — its dead-letter queue, wired to the main queue via an SQS redrive policy.

Key behaviors:

- **Durability** — every message is a row inside Amazon SQS; nothing is lost on process restart.
- **Group ordering** — `--group` on `send` maps to the FIFO `MessageGroupId`, so messages sharing a
  group are delivered one at a time, in order; messages without a group each get a random group so
  they can be processed in parallel.
- **Attempts and dead-lettering** — SQS tracks each message's delivery count itself
  (`ApproximateReceiveCount`). Once the queue's `--maxAttempts` (set at `create` time) is reached,
  SQS moves the message to the dead-letter queue automatically; no explicit step is required on
  `nack`.
- **Leases** — `lease` returns the SQS receipt handle as `leaseId`; `ack` deletes the message by
  that handle, `nack` releases it immediately (visibility timeout 0) for redelivery, and `renew`
  extends the visibility timeout without touching the message otherwise.
- **Credentials, region, endpoint** — resolved entirely by the AWS CLI's own standard mechanism
  (environment variables, named profile, or attached role). Region comes from `AWS_REGION` /
  `AWS_DEFAULT_REGION`; a custom endpoint (e.g. for local testing against ElasticMQ or LocalStack)
  comes from `AWS_ENDPOINT_URL`. `aux4/queue-sqs` never stores or asks for credentials itself.

A few flags accepted by `aux4 queue` for other providers have no effect on `--provider sqs`, because
Amazon SQS already handles the same concern natively: `--owner` (no native lease-owner concept
beyond the receipt handle), `--maxAttempts` on `lease`/`nack`/`dlq` (already enforced by the queue's
own redrive policy), and `--workQueue` (every `--provider sqs` queue is always a durable work queue).

#### Usage

```bash
aux4 queue create --name <name> --provider sqs [--maxAttempts <n>]
aux4 queue delete --name <name> --provider sqs
aux4 queue list --provider sqs
aux4 queue send --name <name> --provider sqs [--group <key>]
aux4 queue lease --name <name> --provider sqs [--visibility <seconds>]
aux4 queue ack --name <name> --provider sqs --leaseId <leaseId>
aux4 queue nack --name <name> --provider sqs --leaseId <leaseId>
aux4 queue renew --name <name> --provider sqs --leaseId <leaseId> [--visibility <seconds>]
aux4 queue dlq --name <name> --provider sqs
```

--name         Queue name
--provider     Set to `sqs` to use this plugin's backend (default across `aux4 queue` is `memory`); `--storage sqs` is still accepted as an alias
--maxAttempts  Maximum delivery attempts before dead-lettering; 0 (default) means unlimited, which SQS approximates with its own maximum of 1000
--group        Group key for one-at-a-time delivery; a random group is used per message when omitted
--visibility   Visibility timeout in seconds (default: 30)
--leaseId      The receipt handle returned by `queue lease` (also accepted as `--id`)

#### Example

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:9324

aux4 queue create --name jobs --provider sqs --maxAttempts 3
echo '{"task":"build"}' | aux4 queue send --name jobs --provider sqs
aux4 queue lease --name jobs --provider sqs --visibility 30
```

```text
{"id":"34b69a0a-...","leaseId":"34b69a0a-...#4a77763f-...","group":"6A286799-...","attempts":1,"body":{"task":"build"}}
```
