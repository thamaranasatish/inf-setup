### FILE: execution-plan/12-phase-8c-vm8-vm9-elk-grafana-install.md

# Phase 8c — VM8/VM9 ELK + Grafana Install

## Objective
Install Elasticsearch, Logstash, Kibana, and Grafana on VM8 and VM9 per official vendor docs.

## Steps

1. Install Elasticsearch using the official Elastic Stack RPM/DEB install guide.
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/starting-elasticsearch.html

2. Install Logstash.
   - Official doc: https://www.elastic.co/guide/en/logstash/current/installing-logstash.html

3. Install Kibana.
   - Official doc: https://www.elastic.co/guide/en/kibana/current/install.html

4. Install Grafana OSS/Enterprise per official RPM/DEB install guide.
   - Official doc: https://grafana.com/docs/grafana/latest/setup-grafana/installation/

5. Configure Grafana data sources for Elasticsearch and Prometheus as required.
   - Official doc: https://grafana.com/docs/grafana/latest/datasources/
   - Official doc: https://grafana.com/docs/grafana/latest/datasources/elasticsearch/

6. Configure Elasticsearch clustering (single-node in DEV) and security settings per official guide.
   - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-stack-security.html

## Validation / Expected Results

- `curl -s http://<elasticsearch-host>:9200/_cluster/health` returns `green` or `yellow` per DEV topology.
  - Official doc: https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html
- Kibana UI reachable and connected to Elasticsearch.
- Grafana UI reachable and data sources healthy.
  - Official doc: https://grafana.com/docs/grafana/latest/

## Exit Criteria

- ELK + Grafana reachable on VM8 and VM9 for DEV workload telemetry.
