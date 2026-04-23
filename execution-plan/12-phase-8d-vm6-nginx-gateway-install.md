### FILE: execution-plan/12-phase-8d-vm6-nginx-gateway-install.md

# Phase 8d — VM6 F5 NGINX API Gateway Install

## Objective
Install and configure F5 NGINX (Plus or OSS per client license) as the API gateway on VM6.

## Steps

1. Install NGINX Plus on RHEL per the official F5 NGINX installation guide.
   - Official doc: https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-plus/

2. (Alternative, if OSS is approved) Install NGINX OSS per the official install guide.
   - Official doc: https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/

3. Configure API gateway functionality and routing per the official NGINX API gateway guide.
   - Official doc: https://docs.nginx.com/nginx/deployment-guides/load-balance-third-party/api-gateway/

4. Configure TLS termination, upstream pools, and health checks.
   - Official doc: https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/
   - Official doc: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-health-check/

5. Configure access logs and forward to VM8/VM9 (ELK) per Elastic’s ingest guidance.
   - Official doc: https://docs.nginx.com/nginx/admin-guide/monitoring/logging/

## Validation / Expected Results

- `nginx -t` passes and `systemctl status nginx` reports `active (running)`.
- `curl -sk https://<nginx-host>/health` returns the expected healthy response.
- Upstream health checks report healthy members for each route.

## Exit Criteria

- VM6 NGINX gateway serving approved routes with TLS and logging enabled.
