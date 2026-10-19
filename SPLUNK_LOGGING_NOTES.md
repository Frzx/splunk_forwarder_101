# Doris MCP logs in Splunk: learning notes

## 1. What we built

We run Doris MCP Server and a Splunk Universal Forwarder together. A separate
Splunk Enterprise container receives, indexes, and searches their log events.

```text
MCP Inspector
    |
    | HTTP: http://127.0.0.1:3000/mcp
    v
Doris MCP Server -------- SQL connection --------> Apache Doris
    |
    | Writes application and audit log files
    v
Docker named volume: mcp-logs
    |
    | Forwarder monitors the files for new entries
    v
Splunk Universal Forwarder
    |
    | Splunk forwarding protocol over TCP
    | Destination: host.docker.internal:9997
    v
Windows host port 9997 -> Splunk container port 9997
    |
    v
Splunk Enterprise -> index=main -> Search & Reporting
```

The shared filesystem is between MCP and its forwarder only. Splunk receives
events over the network; it does not mount the MCP log volume.

These are **Doris MCP Server logs**, not the Apache Doris database's FE/BE logs.
The database would need its own log collection configuration.

## 2. What each component does

| Component | Responsibility |
| --- | --- |
| Apache Doris | Stores data and executes SQL requested through MCP. |
| Doris MCP Server | Exposes database tools over MCP and writes its own logs. |
| Universal Forwarder | Watches local files, tracks how far it has read, and forwards new events. |
| Splunk Enterprise | Receives events, assigns/parses metadata, indexes data, and runs searches. |
| MCP Inspector | Connects to MCP and invokes tools; it is not the log collector. |

Doris MCP does not automatically discover or send logs to the Universal
Forwarder. It does not need to know that the forwarder exists. The forwarder is
configured independently to read the files MCP already produces.

We found no supported built-in Splunk destination or HEC-token configuration in
the official Doris MCP configuration reference. MetricFlow settings concern
semantic business metrics, not exporting server logs to Splunk.

## 3. Files in this project

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | Runs MCP, its forwarder, and two initialization services. |
| `deployment/Dockerfile` | Builds the MCP image with `doris-mcp-server==0.6.1`. |
| `.env` | Holds the existing Doris connection settings used by MCP. |
| `deployment/splunk/forwarder/local/inputs.conf` | Defines which log files to collect and their metadata. |
| `deployment/splunk/forwarder/local/outputs.conf` | Defines the receiving Splunk destination. |

The separate Splunk Enterprise Compose project is managed independently.
Files under `deployment/splunk/receiver/` are optional examples; the current MCP
Compose file does not install them into that separate Splunk instance.

## 4. How file logging works

The MCP container has these settings:

```yaml
LOG_FILE_PATH: /app/logs/server.log
LOG_LEVEL: INFO
ENABLE_AUDIT: "true"
```

For the installed **MCP version 0.6.1**, `LOG_FILE_PATH` is treated as a file
path, and the implementation uses its **parent directory**, `/app/logs`.
MCP generates its own filenames; it does not write everything to `server.log`.

The files include:

```text
/app/logs/
├── doris_mcp_server_all.log
├── doris_mcp_server_audit.log
├── doris_mcp_server_debug.log
├── doris_mcp_server_info.log
├── doris_mcp_server_warning.log
├── doris_mcp_server_error.log
└── doris_mcp_server_critical.log
```

The directory name is our choice. What matters is that MCP can write there and
the forwarder can read the same files. Audit logging being enabled does not
guarantee audit events for every operation; the audit file may remain empty.
Messages written only to stdout/stderr are not collected by these file monitors.

## 5. One volume, two container paths

The same named volume is mounted at different paths:

| Container | Mount | Access |
| --- | --- | --- |
| MCP | `mcp-logs:/app/logs` | Read/write |
| Forwarder | `mcp-logs:/mnt/doris-logs:ro` | Read-only |

Therefore these paths refer to the same underlying file:

```text
MCP:       /app/logs/doris_mcp_server_all.log
Forwarder: /mnt/doris-logs/doris_mcp_server_all.log
```

`ro` means read-only. The forwarder needs to read the application logs, not modify
them. Its own state and checkpoints are written to separate persistent volumes.

## 6. What inputs.conf means

```ini
[monitor:///mnt/doris-logs/doris_mcp_server_all.log*]
disabled = 0
index = main
sourcetype = doris:mcp
host = doris-mcp-server

[monitor:///mnt/doris-logs/doris_mcp_server_audit.log*]
disabled = 0
index = main
sourcetype = doris:mcp:audit
host = doris-mcp-server
```

| Setting | Meaning |
| --- | --- |
| `monitor://...` | Monitor files at this path. The wildcard also matches rotation backups. |
| `disabled = 0` | Enable this input. |
| `index = main` | Store events in Splunk's existing `main` index. |
| `sourcetype` | Label the format/type of events for parsing and searching. |
| `host` | Assign a stable source-host label to the events. |

Splunk also records `source`, which identifies the monitored file path. In the
working search, it was `/mnt/doris-logs/doris_mcp_server_all.log`.

We collect the combined application log and the separate audit log. Collecting
the per-level files as well would duplicate application messages already present
in the combined log.

## 7. What outputs.conf means

```ini
[tcpout]
defaultGroup = doris_receiver
indexAndForward = false

[tcpout:doris_receiver]
server = host.docker.internal:9997
useACK = true
```

- `defaultGroup` selects the named destination group below it.
- `server` specifies the receiver's hostname and TCP port.
- `indexAndForward = false` avoids local indexing as part of this forwarding setup.
- `useACK = true` enables indexer acknowledgments for delivery tracking. It does
  not guarantee exactly-once delivery or eliminate every possible duplicate.

On Docker Desktop, `host.docker.internal` reaches the Windows host. Docker's
published port then routes the connection into the separate Splunk container.

Using `localhost` inside the forwarder would refer to the forwarder container
itself, not the separate Splunk container.

## 8. Why Splunk needs port 9997

Two distinct configurations are necessary for this route:

1. Publish `9997:9997` in the separate Splunk Compose file.
2. Enable receiving on port 9997 in Splunk Web:
   **Settings -> Forwarding and receiving -> Configure receiving**.

Publishing a port exposes it through Docker; enabling the receiver tells Splunk
to actually listen for forwarded data.

| Port | Purpose | Used by our log pipeline? |
| --- | --- | --- |
| 8000 | Splunk Web UI | Used to search the results. |
| 8089 | Splunk management API, normally HTTPS | Not the forwarding endpoint. |
| 514/UDP | Syslog, when configured | No. |
| 8088 | HTTP Event Collector, when enabled | No. |
| 9997/TCP | Conventional Splunk forwarding receiver port | Yes. |

HEC is another valid production ingestion approach. It uses an HTTP-capable
sender and a HEC token. Our standard Universal Forwarder setup instead uses
Splunk's forwarding protocol on 9997; no HEC token is needed for this pipeline.

A mapping such as `127.0.0.1:8000:8000` restricts the published port to the host's
loopback interface. `8000:8000` publishes on all host interfaces by default.

## 9. Startup and persistent storage

The Compose file has two initialization services:

### log-permissions

Runs as root, assigns the shared log directory to MCP's application user, and
sets permissions so the forwarder can read it. It then exits successfully.
MCP itself continues to run as non-root `appuser`.

### forwarder-config

Seeds the forwarder's configuration volume from the image's supplied defaults,
copies our custom configuration app into it, assigns ownership to `splunk`, and
exits. The source configuration is mounted read-only outside the installation;
the installed copy is writable for Splunk provisioning.

```text
log-permissions succeeds ---------> MCP starts
             |
             +-------------------> Forwarder starts
forwarder-config succeeds --------> Forwarder starts
```

`Exited (0)` is expected for both initialization services. A container being
reported as running or healthy does not, by itself, prove that logs are indexed.

| Volume | What it preserves |
| --- | --- |
| `mcp-logs` | MCP's application and audit log files. |
| `forwarder-etc` | Forwarder configuration and instance settings. |
| `forwarder-data` | Forwarder runtime data, internal logs, and file-reading checkpoints. |

The checkpoints help the forwarder resume reading after recreation. Avoid
`docker compose down -v` when you want to preserve these volumes. Recreating the
same Compose project does not reset its persistent volumes.

## 10. Commands to run and inspect the setup

Run these from the MCP project directory:

```powershell
# Validate the Compose configuration.
docker compose config --quiet

# Build and start the services in the background.
docker compose up -d --build

# Show running and initialization services.
docker compose ps -a

# Show container startup output.
docker compose logs --tail 100 splunk-forwarder
docker compose logs --tail 100 doris-mcp-server

# Check files written by MCP.
docker compose exec doris-mcp-server ls -l /app/logs

# Inspect effective forwarder configuration as the Splunk service user.
docker compose exec --user splunk splunk-forwarder /opt/splunkforwarder/bin/splunk btool inputs list --debug
docker compose exec --user splunk splunk-forwarder /opt/splunkforwarder/bin/splunk btool outputs list --debug
```

The Dockerfile already creates `/app/logs` with application-user ownership.
The initialization service additionally repairs ownership on an existing volume.

## 11. Finding the logs in Splunk

Open **Search & Reporting** and enter this in the main search box:

```spl
index=main host="doris-mcp-server"
```

Select historical **All time** and click Search. Start with historical search
when validating ingestion; **All time (real-time)** is a different search mode.

Application events:

```spl
index=main host="doris-mcp-server" sourcetype="doris:mcp"
```

Audit events, if any have been generated:

```spl
index=main host="doris-mcp-server" sourcetype="doris:mcp:audit"
```

Count events by type:

```spl
index=main host="doris-mcp-server"
| stats count by sourcetype
```

Find warning and error text:

```spl
index=main host="doris-mcp-server" ("[WARNING]" OR "[ERROR]" OR "[CRITICAL]")
```

During verification, Splunk showed **334 events**, including tool calls such as
`get_db_list`. That count was a snapshot, not a fixed expected total. New events
appear as MCP performs operations and the forwarder sends additional log entries.

## 12. Problems we encountered

| Symptom | Cause and resolution |
| --- | --- |
| `services.splunk must be a mapping` | The image was placed directly after `splunk:`. Put it under an indented `image:` key. |
| Splunk startup variables not recognized | Corrected `SPLUNK_PASWORD` to `SPLUNK_PASSWORD` and `SPLUNK_GENERAT_TERMS` to `SPLUNK_GENERAL_TERMS`. |
| Invalid volume declarations or health-check field | Named volume declarations need trailing colons; use `interval`, not `intervel`. |
| Permission denied under `/app/logs` | The named volume was root-owned. Added the ownership initialization service. Changing the image alone does not repair an existing volume. |
| Inspector showed `fetch failed` | Its saved endpoint used `htttp://` instead of `http://`. Corrected it to `http://127.0.0.1:3000/mcp`. |
| No events despite a running forwarder container | Provisioning was stuck on recursive ownership changes against a read-only app mount. Staged and copied configuration into the writable persistent configuration volume instead. |
| Search remained empty during validation | Used a historical All time search after fixing forwarding; verified actual indexed events rather than relying on container health. |

## 13. Moving the same design to two machines

On the application machine, run MCP and its forwarder with access to local log
files. On the observability machine, run Splunk with receiving enabled. Change
`outputs.conf` from `host.docker.internal:9997` to the receiver's reachable hostname
and port. The filesystem remains local to the application machine.

This learning lab uses plain TCP. For a real deployment, configure TLS and
certificate verification, restrict receiver network access, pin tested image
versions, and supply passwords through managed configuration rather than the
example credentials currently in Compose. The MCP non-loopback anonymous-access
override is also a local-lab choice; configure authentication for remote access.

## References

- [Doris MCP configuration reference](https://github.com/apache/doris-mcp-server/blob/master/docs/reference/configuration.md)
- [Doris MCP 0.6.1 package documentation](https://pypi.org/project/doris-mcp-server/0.6.1/)
- [Official Splunk Docker examples](https://github.com/splunk/docker-splunk/blob/develop/docs/EXAMPLES.md)
