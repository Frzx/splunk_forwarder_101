# MCP and Universal Forwarder

The root docker-compose.yml runs MCP and its Universal Forwarder. Splunk runs
separately. MCP writes to /app/logs in the mcp-logs volume; the forwarder reads
it at /mnt/doris-logs and sends events to host.docker.internal:9997. On Docker
Desktop this reaches the Windows host's published Splunk receiving port.

## Start

Keep existing Doris settings in the root .env and add:

```dotenv
SPLUNK_UF_PASSWORD=<choose-a-strong-forwarder-admin-password>
```

This password is for the forwarder's own admin account; it need not match Splunk.
Ensure the separate Splunk container publishes 9997:9997 and receiving is enabled
on 9997 under Settings > Forwarding and receiving.

From the repository root:

```powershell
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

The configured startup flags accept Splunk's license and General Terms.
Provisioning may take several minutes. In Splunk Search, select All time:

```spl
index=main host=doris-mcp-server
```

## Configuration

- forwarder/local/inputs.conf monitors application and audit logs, including
  rotation backups. Per-level copies are excluded to avoid duplicate events.
- forwarder/local/outputs.conf sets the destination and enables indexer ACKs.
- forwarder-config seeds the persistent configuration and copies the custom app
  into it before startup. Config is staged outside the Splunk installation so
  the image's recursive ownership task can update it successfully.
- MCP's Dockerfile creates a writable /app/logs directory for its application user.
- The log-permissions initialization service also repairs ownership on existing
  log volumes before MCP and the forwarder start. It exits after succeeding;
  an Exited (0) status for that service is expected.
- forwarder-data persists file-reading checkpoints across container recreation.
- receiver/ contains optional examples; this Compose file does not mount them or
  manage your separate Splunk instance.

These are MCP application/audit logs, not Doris database FE/BE logs. Console-only
messages are not collected by the file monitors.

## Inspect

```powershell
docker compose logs --tail 100 splunk-forwarder
docker compose exec doris-mcp-server ls -l /app/logs
docker compose exec splunk-forwarder /opt/splunkforwarder/bin/splunk btool inputs list --debug
docker compose exec splunk-forwarder /opt/splunkforwarder/bin/splunk btool outputs list --debug
```

This lab uses plain TCP. For a real remote server, change outputs.conf to its
hostname and configure TLS with certificate verification and restricted receiver
access. Keep volumes when stopping to preserve logs and checkpoints (omit down -v).
Delivery is verified only when actual events appear in Splunk.
