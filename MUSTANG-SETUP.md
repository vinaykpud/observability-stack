# Observability stack against a local Mustang node

Run this observability stack with **Data Prepper writing telemetry to a locally
running Mustang (OPTIMIZED / parquet) node** — in addition to the bundled Lucene
cluster, which the OSD UI reads.

This is a *dual-write* setup: every log / span / service-map record goes to **both**
- **Mustang** (parquet, `otel-*-plain`) — queried via **PPL** only, and
- **Lucene** (the bundled GENERAL cluster, `otel-v1-apm-span` / `logs-otel-v1` / `otel-v2-apm-service-map`) — what OSD Trace Analytics / Discover / dashboards read.

Metrics go to Cortex (Prometheus remote-write), same as the stock stack.

## What this setup adds (the branch payload)

```
docker-compose.dp-mustang.yml                              # Mustang override for data-prepper + dashboards
docker-compose.local-env-fixes.yml                        # 9201 remap, envoy DNS resolver, alertmanager gossip, memory
docker-compose/data-prepper/pipelines.mustang.yaml        # the dual-write DP pipeline
docker-compose/data-prepper/mustang-otel-logs-template.json         # parquet index templates DP installs itself
docker-compose/data-prepper/mustang-otel-spans-template.json
docker-compose/data-prepper/mustang-otel-service-map-template.json
mustang-bridge-fwd.py                                     # loopback -> docker-bridge forwarder (see step 1)
MUSTANG-SETUP.md                                          # this file
```

## Prerequisites

**1. A Mustang dev-tree node** on `127.0.0.1:9200` (`./gradlew run`) with the
**sandbox / parquet plugins** (pluggable dataformat) **and the SQL plugin** (for PPL).
This is the node the stack points at.

**2. Docker (or Finch) + docker compose**, Git, and enough memory — the OTel demo
needs ~2 GB on top of the core stack. The upstream installer checks all of this.

**3. Install the base stack so `.env` + images exist.** This repo is the upstream
OpenSearch observability-stack; use its installer, then apply the Mustang overrides
in the steps below.

```bash
# clone your branch that carries the Mustang files (or the upstream + copy them in)
git clone <this-branch-url> observability-stack && cd observability-stack

./install.sh --simulate     # dry run: prints requirements + what it will do
./install.sh                # checks Docker/memory, pulls/builds images, writes .env,
                            # and brings up the STOCK stack (Step 2 re-applies with Mustang)
```
Full quickstart, requirements, and options (`--skip-pull`, Finch, etc.):
<https://github.com/opensearch-project/observability-stack#-quickstart>

**4. Confirm `.env` is set for this variant** (the installer writes most of these;
uncomment the OTel demo, which generates the telemetry):
```bash
grep -E '^(COMPOSE_PROFILES|INCLUDE_COMPOSE_LOCAL_OPENSEARCH|INCLUDE_COMPOSE_LOCAL_OPENSEARCH_DASHBOARDS|INCLUDE_COMPOSE_OTEL_DEMO)=' .env
```
should show:
```
COMPOSE_PROFILES=local-backends
INCLUDE_COMPOSE_LOCAL_OPENSEARCH=docker-compose.local-opensearch.yml
INCLUDE_COMPOSE_LOCAL_OPENSEARCH_DASHBOARDS=docker-compose.local-opensearch-dashboards.yml
INCLUDE_COMPOSE_OTEL_DEMO=docker-compose.otel-demo.yml        # the webstore + Locust load generator
```
UI credentials also live in `.env`: `grep -E '^OPENSEARCH_(USER|PASSWORD)=' .env`.

> `./install.sh` starts the **stock** stack (Data Prepper → Lucene only). Step 2
> below re-applies the compose set **with** the Mustang overrides, which recreates
> `data-prepper` + `opensearch-dashboards` to point at your Mustang node. If you'd
> rather not start the stock stack first, you can still run `./install.sh --simulate`
> only for the checks, ensure `.env` exists, then jump straight to Step 2.

## Step 1 — make the Mustang node reachable from the containers

The dev-tree node binds `127.0.0.1` only. `docker-compose.dp-mustang.yml` adds
`extra_hosts: host.docker.internal:host-gateway` so containers resolve
`host.docker.internal` to the docker0 gateway — but something must **listen** there.
Easiest (no node restart):

```bash
python3 mustang-bridge-fwd.py "$(ip -4 addr show docker0 | grep -oP 'inet \K[\d.]+')"
# forwards <docker0-gateway>:9200 -> 127.0.0.1:9200 ; keep this process running
```

Alternative (permanent, but a node restart **wipes the testclusters data dir** unless
you also pass `--preserve-data`):
```
./gradlew run ... -Dtests.opensearch.http.host=127.0.0.1,<docker0-gateway> --preserve-data
```

> If this forwarder dies, Data Prepper loses the Mustang node (Lucene keeps working).

## Step 2 — bring up the stack

```bash
cd <this repo>
unset OPENSEARCH_HOST OPENSEARCH_PORT OPENSEARCH_PROTOCOL OPENSEARCH_USER OPENSEARCH_PASSWORD \
      INCLUDE_COMPOSE_LOCAL_OPENSEARCH INCLUDE_COMPOSE_LOCAL_OPENSEARCH_DASHBOARDS
docker compose -f docker-compose.yml \
               -f docker-compose.local-env-fixes.yml \
               -f docker-compose.dp-mustang.yml up -d
```

This starts the Lucene GENERAL OpenSearch + OpenSearch Dashboards + Cortex +
otel-collector + data-prepper (with the Mustang pipeline and template mounts) +
the demo apps and load generator.

## Step 3 — what happens automatically (no manual index/template steps)

- Data Prepper **installs the three parquet templates** (mounted into
  `/usr/share/data-prepper/templates/`) and **auto-creates**
  `otel-{logs,spans,service-map}-plain` on the Mustang node on first write.
- Data Prepper **dual-writes** to the Lucene cluster for the OSD UI.
- The OSD init container creates the workspace, the `local_cluster` data source,
  index patterns, and dashboards.

## Step 4 (optional) — register `mustang-local` as an OSD data source

Only needed to run **PPL against Mustang from the UI** (Query Workbench / Explore).
The init only creates `local_cluster`, so this is manual per fresh stack:

```bash
PW='<OPENSEARCH_INITIAL_ADMIN_PASSWORD from .env>'
OSD(){ docker exec opensearch-dashboards curl -s -u "admin:$PW" -H 'osd-xsrf:true' -H 'content-type:application/json' "$@"; }
WS=$(OSD -XPOST localhost:5601/api/workspaces/_list -d '{}' \
      | python3 -c 'import json,sys;print(json.load(sys.stdin)["result"]["workspaces"][0]["id"])')
OSD -XPOST localhost:5601/api/saved_objects/data-source/mustang-local -d \
  '{"attributes":{"title":"mustang-local (OPTIMIZED / parquet)","endpoint":"http://host.docker.internal:9200","auth":{"type":"no_auth"},"dataSourceEngineType":"OpenSearch","dataSourceVersion":"3.9.0"}}'
OSD -XPOST localhost:5601/api/workspaces/_associate -d \
  "{\"workspaceId\":\"$WS\",\"savedObjects\":[{\"type\":\"data-source\",\"id\":\"mustang-local\"}]}"
```

## Step 5 — verify

```bash
curl -XPOST localhost:9200/otel-*-plain/_refresh
curl 'localhost:9200/_cat/indices/otel-*-plain?v'        # docs climbing on Mustang
docker logs data-prepper --since 60s | grep -c ERROR     # expect 0
# UI:            http://localhost:5601   (admin / <pw>)
# demo + loadgen: http://localhost:8080
```

## Gotchas

- **Forwarder must stay alive** (step 1) — otherwise DP can't reach Mustang.
- **Parquet indices:** no auto-refresh (use `_refresh`); `_count` / `_search` are
  unsupported (`Cannot apply function on indexer class DataFormatAwareEngine …`) —
  use `_cat/indices` for counts and **PPL** for reads.
- **By default the UI renders off Lucene, not Mustang.** The auto-created index
  patterns, Trace Analytics, and the demo dashboards all read the Lucene dual-write.
  To view the **Mustang** (parquet) data in the UI, **manually create datasets /
  index patterns against the `mustang-local` data source** for the
  `otel-spans-plain`, `otel-logs-plain`, and `otel-service-map-plain` indices, then
  query them via **PPL** (Query Workbench / Explore).
- **`mustang-local` is a manual step** each fresh stack (step 4), until it's baked
  into the OSD init script.
