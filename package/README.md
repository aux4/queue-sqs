# aux4/queue-sqs

Amazon SQS provider plugin for [aux4/queue](https://hub.aux4.io/r/public/packages/aux4/queue)'s
durable work-queue mode. Installing this package adds `sqs` as a `--provider` choice to the
`aux4 queue` commands you already use — `create`, `delete`, `list`, `send`, `lease`, `ack`, `nack`,
`renew`, `dlq` — so a script written against `aux4 queue`'s memory or repository provider can switch
to a durable, managed Amazon SQS backend by changing one flag, with no other code changes.

This package does not add a new top-level command. Once installed alongside `aux4/queue`, use
`aux4 queue <command> --provider sqs ...` exactly like you would with `--provider memory` or
`--provider repository`. The older `--provider sqs` spelling still works as a backward-compatible
alias for `--provider sqs`.

## Installation

```bash
aux4 aux4 pkger install aux4/queue-sqs
```

This also installs `aux4/queue` (a declared dependency) if it isn't already present.

## How it works

Every `aux4 queue ... --provider sqs` command shells out to the AWS CLI (`aws sqs ...`). There is no
server process and no local database — Amazon SQS itself is the durable store. Each queue created
with `--provider sqs` is backed by two real SQS FIFO queues:

- `<name>.fifo` — the queue itself.
- `<name>-dlq.fifo` — its dead-letter queue, wired up automatically via an SQS redrive policy.

FIFO queues are used throughout so that `--group` delivers messages for the same group in order and
one at a time, matching `aux4 queue`'s own work-queue semantics. Within a group, each `queue-sqs`
message gets a unique deduplication id, so sending the same body twice never silently drops a
message.

## Credentials, region, and endpoint

`aux4/queue-sqs` never stores or asks for AWS credentials. It relies entirely on the AWS CLI's own
standard resolution: environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
`AWS_SESSION_TOKEN`), a named profile (`AWS_PROFILE`), or an attached role (EC2/ECS/Lambda instance
role). Set these up the same way you would for any other `aws` CLI usage.

Because `aux4 queue`'s own commands don't carry provider-specific flags beyond `--provider`, region
and endpoint overrides are also read from the AWS CLI's standard environment variables rather than
an aux4 flag:

- `AWS_REGION` / `AWS_DEFAULT_REGION` — AWS region.
- `AWS_ENDPOINT_URL` — override the SQS endpoint, e.g. to point at a local SQS-compatible server
  (ElasticMQ, LocalStack) for testing:

  ```bash
  export AWS_ENDPOINT_URL=http://localhost:9324
  export AWS_ACCESS_KEY_ID=test
  export AWS_SECRET_ACCESS_KEY=test
  export AWS_DEFAULT_REGION=elasticmq

  aux4 queue create --name jobs --provider sqs --maxAttempts 3
  ```

## IAM permissions

The credentials you configure need at least the following SQS actions, scoped to the queues this
package manages (`<name>.fifo` and `<name>-dlq.fifo`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:CreateQueue",
        "sqs:DeleteQueue",
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes",
        "sqs:SetQueueAttributes",
        "sqs:ListQueues",
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:*:*:*"
    }
  ]
}
```

## Usage

### Create a work queue

```bash
aux4 queue create --name jobs --provider sqs --maxAttempts 5
```

Creates `jobs.fifo` and `jobs-dlq.fifo`, with a redrive policy that moves a message to the
dead-letter queue after 5 delivery attempts. `--maxAttempts 0` (the default) is treated as
"unlimited", which SQS approximates with its own maximum of 1000 attempts.

### Send, lease, ack, nack

```bash
echo '{"task":"build"}' | aux4 queue send --name jobs --provider sqs
echo '{"task":"reply"}' | aux4 queue send --name jobs --provider sqs --group chat-42   # same group = one at a time
```

```bash
aux4 queue lease --name jobs --provider sqs --visibility 30
# {"id":"34b69a0a-...","leaseId":"34b69a0a-...#4a77763f-...","group":"","attempts":1,"body":{"task":"build"}}
```

`lease` returns the next eligible message and holds it under a visibility timeout: while the lease
is active, no other `lease` call can pick it up — and when the message has a `--group`, no sibling
in the same group either. If nothing is eligible, `lease` prints `{}` and exits non-zero.

```bash
aux4 queue ack --name jobs --provider sqs --leaseId '34b69a0a-...#4a77763f-...'    # done — remove it
aux4 queue nack --name jobs --provider sqs --leaseId '34b69a0a-...#4a77763f-...'   # failed — redeliver now
```

`nack` releases the message for immediate redelivery. Amazon SQS tracks the delivery count itself;
once the queue's `--maxAttempts` (set at `create` time) is reached, SQS moves the message to the
dead-letter queue on its own — no extra step needed.

### Renew a lease

If processing a message takes longer than the `--visibility` window originally leased, extend it
without losing the lease or acknowledging the message yet:

```bash
aux4 queue renew --name jobs --provider sqs --leaseId '34b69a0a-...#4a77763f-...' --visibility 60
```

### Dead-letter queue

```bash
aux4 queue dlq --name jobs --provider sqs
# {"id":"...","attempts":6,"group":"chat-42","body":{"task":"reply"}}
```

`dlq` peeks at up to 10 dead-letter messages (SQS's own per-call maximum) without removing them.
Amazon SQS has no API to browse a queue's full contents, so a very large dead-letter queue may need
more than one call to see everything.

### List and delete

```bash
aux4 queue list --provider sqs
aux4 queue delete --name jobs --provider sqs
```

`list` returns every queue at the configured endpoint, including dead-letter queues. `delete`
removes both `<name>.fifo` and `<name>-dlq.fifo`.

## Compatibility with aux4/queue

Every flag used above (`--group`, `--visibility`, `--leaseId`/`--id`, `--maxAttempts`) is the exact
same flag `aux4 queue` already accepts for `--provider memory` and `--provider repository`, so a
script only needs to change `--provider` to move between backends. A few flags accepted elsewhere
have no effect on `--provider sqs` because Amazon SQS already handles the same concern natively:

- `--owner` (on `lease`) — SQS has no concept of a lease owner beyond the receipt handle it already
  returns as `leaseId`.
- `--maxAttempts` (on `lease`, `nack`, `dlq`) — already enforced by the queue's own SQS redrive
  policy, configured once at `create` time.
- `--workQueue` (on `create`) — every `--provider sqs` queue is always a durable work queue.
