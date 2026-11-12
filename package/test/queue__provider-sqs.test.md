# aux4 queue work queue (sqs storage)

These tests run against [ElasticMQ](https://github.com/softwaremill/elasticmq), an SQS-API-compatible
server, via `AWS_ENDPOINT_URL`. ElasticMQ was used instead of LocalStack because LocalStack now
requires an auth token even for the SQS service under its community license, while ElasticMQ needs
no account/token at all for a local, disposable test instance. The AWS credentials below are dummy
values ElasticMQ ignores; no real AWS account or network access is used.

Requires `aux4/queue-sqs` installed alongside `aux4/queue` (adds `--provider sqs`, with `--storage sqs`
kept as a working backward-compatible alias), Docker, and the AWS CLI.

````beforeAll
docker rm -f queue-sqs-test-elasticmq >/dev/null 2>&1
docker run -d --name queue-sqs-test-elasticmq -p 19781:9324 softwaremill/elasticmq-native >/dev/null
sleep 3
````

````afterAll
docker rm -f queue-sqs-test-elasticmq >/dev/null 2>&1
````

````beforeEach
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
aux4 queue create --name wq --provider sqs --maxAttempts 2 >/dev/null
````

````afterEach
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
aux4 queue delete --name wq --provider sqs >/dev/null 2>&1
````

### should lease a sent message

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"hello"}' | aux4 queue send --name wq --provider sqs >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30 | jq '{attempts, body}'
````

````expect:json
{
  "attempts": 1,
  "body": {
    "task": "hello"
  }
}
````

Note: `group` is omitted from the comparison above -- when `--group` is not given, `send` assigns a
random group per message (so independent messages can be delivered in parallel instead of
serializing behind one shared default group).

### should not lease the same message again while its lease is active

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"hello"}' | aux4 queue send --name wq --provider sqs >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30 >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30
echo "exit=$?"
````

````expect
{}
exit=4
````

### should ack a leased message, removing it for good

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"once"}' | aux4 queue send --name wq --provider sqs >/dev/null
LEASE=$(aux4 queue lease --name wq --provider sqs --visibility 30)
LID=$(echo "$LEASE" | jq -r '.leaseId')
aux4 queue ack --name wq --provider sqs --leaseId "$LID"
aux4 queue lease --name wq --provider sqs --visibility 30
echo "exit=$?"
````

````expect
{}
exit=4
````

### should nack a leased message for immediate redelivery, bumping attempts

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"retry"}' | aux4 queue send --name wq --provider sqs >/dev/null
LEASE=$(aux4 queue lease --name wq --provider sqs --visibility 30)
LID=$(echo "$LEASE" | jq -r '.leaseId')
aux4 queue nack --name wq --provider sqs --leaseId "$LID"
aux4 queue lease --name wq --provider sqs --visibility 30 | jq '.attempts'
````

````expect
2
````

### should extend a lease with renew, without losing it

````timeout
15000
````

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"longrunning"}' | aux4 queue send --name wq --provider sqs >/dev/null
LEASE=$(aux4 queue lease --name wq --provider sqs --visibility 5)
LID=$(echo "$LEASE" | jq -r '.leaseId')
aux4 queue renew --name wq --provider sqs --leaseId "$LID" --visibility 30
echo "renew-exit=$?"
sleep 6
aux4 queue lease --name wq --provider sqs --visibility 5
echo "still-leased-exit=$?"
````

````expect:partial
renew-exit=0
{}
still-leased-exit=4
````

### should move a message to the dead-letter queue once maxAttempts is reached

````timeout
15000
````

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"task":"doomed"}' | aux4 queue send --name wq --provider sqs >/dev/null
LEASE1=$(aux4 queue lease --name wq --provider sqs --visibility 1)
LID1=$(echo "$LEASE1" | jq -r '.leaseId')
aux4 queue nack --name wq --provider sqs --leaseId "$LID1" >/dev/null
LEASE2=$(aux4 queue lease --name wq --provider sqs --visibility 1)
LID2=$(echo "$LEASE2" | jq -r '.leaseId')
aux4 queue nack --name wq --provider sqs --leaseId "$LID2" >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 1
echo "main-queue-exit=$?"
aux4 queue dlq --name wq --provider sqs | jq '.body'
````

````expect:partial
{}
main-queue-exit=4
{
  "task": "doomed"
}
````

### should deliver only one message at a time within the same group

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"n":1}' | aux4 queue send --name wq --provider sqs --group chat-42 >/dev/null
echo '{"n":2}' | aux4 queue send --name wq --provider sqs --group chat-42 >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30 | jq '.body'
aux4 queue lease --name wq --provider sqs --visibility 30
echo "second-in-group-exit=$?"
````

````expect:partial
{
  "n": 1
}
{}
second-in-group-exit=4
````

### should never let two concurrent leasers get the same message

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
echo '{"n":1}' | aux4 queue send --name wq --provider sqs >/dev/null
echo '{"n":2}' | aux4 queue send --name wq --provider sqs >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30 | jq -r '.id' > /tmp/queue-sqs-test-a.id &
aux4 queue lease --name wq --provider sqs --visibility 30 | jq -r '.id' > /tmp/queue-sqs-test-b.id &
wait
A=$(cat /tmp/queue-sqs-test-a.id)
B=$(cat /tmp/queue-sqs-test-b.id)
if [ "$A" = "$B" ]; then echo "COLLISION"; else echo "DISTINCT"; fi
rm -f /tmp/queue-sqs-test-a.id /tmp/queue-sqs-test-b.id
````

````expect
DISTINCT
````

### should safely handle shell metacharacters in a message body and group

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
rm -f /tmp/queue-sqs-test-pwned
printf '%s' '{"cmd":"it'"'"'s a test; touch /tmp/queue-sqs-test-pwned; echo done `whoami` $(whoami)"}' | aux4 queue send --name wq --provider sqs --group "safe-group" >/dev/null
aux4 queue lease --name wq --provider sqs --visibility 30 | jq '.body'
if [ -f /tmp/queue-sqs-test-pwned ]; then echo "INJECTED"; else echo "SAFE"; fi
````

````expect:partial
{
  "cmd": "it's a test; touch /tmp/queue-sqs-test-pwned; echo done `whoami` $(whoami)"
}
SAFE
````

### should safely handle a queue name containing shell metacharacters

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
rm -f /tmp/queue-sqs-test-name-pwned
aux4 queue create --name 'inj;touch /tmp/queue-sqs-test-name-pwned;echo' --provider sqs --maxAttempts 2 >/dev/null 2>&1
echo "create-exit=$?"
if [ -f /tmp/queue-sqs-test-name-pwned ]; then echo "INJECTED"; else echo "SAFE"; fi
aux4 queue delete --name 'inj;touch /tmp/queue-sqs-test-name-pwned;echo' --provider sqs >/dev/null 2>&1 || true
````

````expect:partial
SAFE
````
