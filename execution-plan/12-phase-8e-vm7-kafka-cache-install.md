### FILE: execution-plan/12-phase-8e-vm7-kafka-cache-install.md

# Phase 8e — VM7 Kafka + Cache Install

## Objective
Install Apache Kafka and the client-approved cache component on VM7.

## Steps

1. Install Apache Kafka using the official Apache Kafka quickstart / operations docs (tarball or packaged install per client).
   - Official doc: https://kafka.apache.org/documentation/#quickstart
   - Official doc: https://kafka.apache.org/documentation/#operations

2. Configure the Kafka broker (`server.properties`), storage layout, log retention, listeners, and advertised listeners.
   - Official doc: https://kafka.apache.org/documentation/#configuration

3. Configure security (TLS / SASL) per official guidance.
   - Official doc: https://kafka.apache.org/documentation/#security

4. Cache component selection: if Redis is chosen, install Redis per the official docs. If another cache is chosen (e.g., Infinispan / Data Grid), follow that product’s official docs.
   - Official doc (Redis): https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-on-linux/
   - Official doc (Red Hat Data Grid / Infinispan): https://docs.redhat.com/en/documentation/red_hat_data_grid
   - Note: the specific cache engine must be confirmed by the client. If the product is not selected, mark as "OFFICIAL DOC NOT FOUND – NEEDS CLIENT CONFIRMATION" for this step.

## Validation / Expected Results

- Kafka broker started and able to create a topic, produce, and consume messages via `kafka-topics.sh` / `kafka-console-producer.sh` / `kafka-console-consumer.sh`.
  - Official doc: https://kafka.apache.org/documentation/#quickstart
- Cache health endpoint returns a healthy response per the selected product’s official documentation.

## Exit Criteria

- Kafka operational; cache operational; both reachable by approved clients.
