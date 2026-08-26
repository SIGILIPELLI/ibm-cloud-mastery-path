# 04 · Event-Driven Architecture (Event Streams / Kafka)

Every service so far has talked over synchronous HTTP. This module
introduces **Event Streams**, IBM Cloud's managed Apache Kafka, for
services that need to react to things happening elsewhere without
polling or tight coupling.

## Why Kafka instead of another HTTP call

An "orders" service calling "inventory," "billing," and "shipping"
synchronously means one slow downstream service makes every order slow,
and adding a fourth consumer means changing the orders service's code.
Publishing an `order.created` event once and letting each downstream
service subscribe independently decouples both the timing and the list of
consumers.

## Provision Event Streams

```bash
ibmcloud resource service-instance-create events-mastery \
  messagehub standard us-south --resource-group-name mastery-path
```

```text
Service instance events-mastery is being created.
OK
```

`standard` plan gives multi-broker replication and mirroring; `lite` (free)
exists only for learning and caps topic count and retention hard enough
that it's unsuitable even for this exercise's throughput tests.

## Create a topic

```bash
ibmcloud es topic-create orders.created \
  --instance events-mastery \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=604800000
```

```text
Topic orders.created has been successfully created.
```

Three partitions lets three consumer instances in a group process in
parallel; partition count can only be *increased* later, never decreased,
so under-provisioning is more forgiving than over-provisioning — but
increasing partition count after producers are live changes key-to-partition
mapping and can reorder events for a given key, so plan the initial count
around expected peak parallelism.

## Get connection credentials

```bash
ibmcloud resource service-key-create events-mastery-key Manager \
  --instance-name events-mastery
```

```text
{
  "kafka_brokers_sasl": [
    "broker-0-xyz.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-1-xyz.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093",
    "broker-2-xyz.kafka.svc09.us-south.eventstreams.cloud.ibm.com:9093"
  ],
  "api_key": "abc123...",
  "user": "token",
  "password": "abc123..."
}
```

## Producer (Node.js, kafkajs)

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'orders-service',
  brokers: process.env.KAFKA_BROKERS.split(','),
  ssl: true,
  sasl: { mechanism: 'plain', username: 'token', password: process.env.KAFKA_API_KEY },
});

const producer = kafka.producer();

async function publishOrderCreated(order) {
  await producer.connect();
  await producer.send({
    topic: 'orders.created',
    messages: [{ key: order.id, value: JSON.stringify(order) }],
  });
}
```

Keying by `order.id` guarantees every event for the same order lands on
the same partition, which guarantees ordering *for that order* — Kafka
never guarantees global ordering across partitions.

## Consumer group

```javascript
const consumer = kafka.consumer({ groupId: 'inventory-service' });

async function run() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'orders.created', fromBeginning: false });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const order = JSON.parse(message.value.toString());
      await reserveInventory(order);
    },
  });
}
```

`fromBeginning: false` starts a new consumer group at the tail — fine for
a live service, wrong for a backfill job, which should use a distinct
`groupId` with `fromBeginning: true` so it doesn't perturb the live
group's committed offsets.

## Mirroring for disaster recovery

Event Streams' `standard`/`enterprise` plans support **Mirror Maker 2** to
replicate topics to a second-region instance, so a regional outage doesn't
lose in-flight events:

```bash
ibmcloud es mirroring-topic-selection-set \
  --instance events-mastery-dr \
  --source-instance events-mastery \
  --topics "orders.*"
```

```text
Mirroring topic selection updated for events-mastery-dr.
```

Mirrored topics get an `-source` suffix on the target side by default —
consumers in the DR region subscribe to `orders.created-source`, not the
original name, which is worth documenting so a failover runbook doesn't
subscribe to a topic that doesn't exist yet.

## Terraform for the instance and topic

```hcl
resource "ibm_resource_instance" "event_streams" {
  name              = "events-mastery"
  service           = "messagehub"
  plan              = "standard"
  location          = "us-south"
  resource_group_id = data.ibm_resource_group.mastery_path.id
}

resource "ibm_event_streams_topic" "orders_created" {
  resource_instance_id = ibm_resource_instance.event_streams.id
  name                  = "orders.created"
  partitions            = 3
  config = {
    "retention.ms" = "604800000"
  }
}
```

```bash
terraform validate
# Success! The configuration is valid.
```

## Gotchas

- **Consumer lag is invisible until you check for it** — always monitor
  `ibmcloud es topic orders.created` and consumer group lag, not just
  producer success; a stuck consumer looks fine to the producer forever.
- **Message size limit** is 1 MB by default per record — large payloads
  (attach a Cloud Object Storage reference instead of embedding a file).
- **Schema drift**: nothing stops a producer from changing a JSON payload
  shape mid-stream; pair Event Streams with a schema registry approach
  (Avro + IBM Event Streams Schema Registry, or at minimum a versioned
  `schema_version` field) once more than one team owns a topic.
- **Idle Kafka clients time out**: SASL_SSL connections behind IBM's load
  balancer close after a period of inactivity — clients need
  reconnect/retry logic, not a bare "connect once" pattern.

## Cheat sheet

| Task | Command |
|---|---|
| Create instance | `ibmcloud resource service-instance-create <n> messagehub standard <region>` |
| Create topic | `ibmcloud es topic-create <name> --instance <n> --partitions <n>` |
| List topics | `ibmcloud es topics --instance <n>` |
| Create service key | `ibmcloud resource service-key-create <n> Manager --instance-name <inst>` |
| Set mirroring topics | `ibmcloud es mirroring-topic-selection-set --instance <dr> --source-instance <src> --topics <pattern>` |
| Delete topic | `ibmcloud es topic-delete <name> --instance <n>` |

## Exercise

1. Create an Event Streams instance and an `orders.created` topic with 3
   partitions.
2. Write a producer that publishes a JSON event keyed by an order ID, and
   a consumer in its own consumer group that reads it back.
3. Explain, in your own words, why keying by order ID preserves
   per-order ordering but not global ordering across all orders.
4. Sketch (Terraform, not applied) a second Event Streams instance in a
   second region and a mirroring topic selection between them.
