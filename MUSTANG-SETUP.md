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

## Step 4 — register `mustang-local` as an OSD data source

Required to query Mustang from the UI (Query Workbench / Explore) and to create
the Mustang data views in Step 4b. The init only creates `local_cluster`, so this
is manual per fresh stack:

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

## Step 4b — create the Mustang data views (traces / logs / service-map)

The OSD init creates data views only for the **Lucene** indices (on
`local_cluster`). For the **Mustang** indices you must create the matching three
against `mustang-local` — mirroring the Lucene ones exactly:

| Mustang index | mirrors Lucene | time field | signalType |
|---|---|---|---|
| `otel-spans-plain`       | `otel-v1-apm-span*`        | `endTime`   | `traces` |
| `otel-logs-plain`        | `logs-otel-v1*`            | `time`      | `logs`   |
| `otel-service-map-plain` | `otel-v2-apm-service-map*` | `timestamp` | *(none)* |

A working Mustang trace source needs these — and the **decisive one is #3**
(verified by isolation: `INDEXES` and `INDEX_PATTERN` both render fine *once the
fields are `searchable: true`*; both fail when `searchable: false`):

1. **`signalType: traces`** — marks it a trace source. (Functionally the dataset
   type, `INDEXES` vs `INDEX_PATTERN`, does **NOT** matter — verified; it was a red
   herring. **By convention we still create these as `INDEX_PATTERN` data views**
   to mirror the Lucene side — the Step 4b script does this — but the rendering
   works either way once #3 is set.)
2. **cached `fields`** — the saved-objects API doesn't fetch fields, so without
   this you get *"Could not locate that index-pattern-field (id: endTime)"*.
3. **`searchable: true` forced on every field** ← **the actual fix.** On a parquet
   index `_field_caps` reports numeric / `date_nanos` / `boolean` fields as
   `searchable: false` (Mustang's Lucene secondary only indexes string types;
   parquet columns aren't `_search`-able). The Traces app gates its **time-link +
   duration-ms rendering on `searchable`**, and `endTime` (date_nanos) +
   `durationInNanos` (long) are exactly the non-searchable types — so the UI stays
   broken until you override them to `true` in the cached field list. This is a
   UI-side override only; it does not make the parquet index actually searchable
   (reads are still PPL).

> You can create the source *any* way (Explore "create dataset", or a data view) —
> what matters is running the `searchable: true` fix on its fields afterward. The
> UI wizards can't set `searchable`, so the API step below is required regardless.

The script below sets `signalType` + fetches fields + forces `searchable: true`
for each data view (reuses `WS` from Step 4):

```bash
python3 - "$WS" <<'PY'
import subprocess, json, sys
PW='<OPENSEARCH_INITIAL_ADMIN_PASSWORD>'; WS=sys.argv[1]
def osd(args, body=None):
    cmd=['docker','exec','-i','opensearch-dashboards','curl','-s','-u',f'admin:{PW}',
         '-H','osd-xsrf:true','-H','content-type:application/json']+args
    return subprocess.run(cmd, input=(body or ''), capture_output=True, text=True).stdout
# id, index, timeField, signalType, displayName
DVS=[("mustang-spans-dv","otel-spans-plain","endTime","traces","Trace Dataset - Mustang"),
     ("mustang-logs-dv","otel-logs-plain","time","logs","Logs Dataset - Mustang"),
     ("mustang-svcmap-dv","otel-service-map-plain","timestamp","","Service-map Dataset - Mustang")]
for did,idx,tf,sig,disp in DVS:
    # 1+2 create the INDEX_PATTERN with signalType
    attrs={"title":idx,"timeFieldName":tf,"displayName":disp}
    if sig: attrs["signalType"]=sig
    osd(['-XPOST','-d','@-',f'localhost:5601/w/{WS}/api/saved_objects/index-pattern/{did}?overwrite=true'],
        json.dumps({"attributes":attrs,"references":[{"id":"mustang-local","type":"data-source","name":"dataSource"}]}))
    # 3 fetch fields, 4 force searchable:true
    fields=json.loads(osd(['-XGET',f'localhost:5601/api/index_patterns/_fields_for_wildcard?pattern={idx}&data_source=mustang-local'])).get("fields",[])
    for f in fields: f["searchable"]=True
    osd(['-XPUT','-d','@-',f'localhost:5601/api/saved_objects/index-pattern/{did}'],
        json.dumps({"attributes":{"fields":json.dumps(fields)}}))
    osd(['-XPOST','-d','@-','localhost:5601/api/workspaces/_associate'],
        json.dumps({"workspaceId":WS,"savedObjects":[{"type":"index-pattern","id":did}]}))
    print(f"{did}: {len(fields)} fields (searchable forced true)")
PY
```

Then in **Explore**, pick these sources under `mustang-local`. Parquet can't serve
`_search`, so data reads go via **PPL** — but with `signalType: traces` + the
fields forced `searchable: true`, the Traces UI renders the trace-detail time link
and ms-formatted duration just like the Lucene side.

> **If you already created the source via the UI** (dataset or data view) and the
> time link / duration are broken, you don't need to recreate it — just run the
> `searchable: true` fix on its existing fields:
> ```bash
> ID=<its saved-object id>   # from Dashboards Mgmt → Index patterns, or the _find API
> python3 - "$ID" <<'PY'
> import subprocess, json, sys
> PW='<OPENSEARCH_INITIAL_ADMIN_PASSWORD>'; ID=sys.argv[1]
> def osd(a,b=None): return subprocess.run(['docker','exec','-i','opensearch-dashboards','curl','-s',
>   '-u',f'admin:{PW}','-H','osd-xsrf:true','-H','content-type:application/json']+a,
>   input=(b or ''),capture_output=True,text=True).stdout
> d=json.loads(osd(['-XGET',f'localhost:5601/api/saved_objects/index-pattern/{ID}']))
> flds=json.loads(d["attributes"]["fields"])
> for f in flds: f["searchable"]=True
> print(osd(['-XPUT','-d','@-',f'localhost:5601/api/saved_objects/index-pattern/{ID}'],
>   json.dumps({"attributes":{"fields":json.dumps(flds)}}))[:120])
> PY
> ```

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
  To view the **Mustang** (parquet) data in the UI, create the `mustang-local`
  data views (Step 4b) and query them via **PPL** (Query Workbench / Explore).
- **Use data views, not ad-hoc datasets, for Mustang (Step 4b).** The Explore
  "create dataset → pick index" flow makes a `type: INDEXES` object → the Traces
  app won't render the trace-detail time link and shows raw (un-ms'd) duration.
  A working trace source needs `type: INDEX_PATTERN` **and** `signalType: traces`,
  which only the Step 4b API calls set (no UI wizard sets both).
- **`mustang-local` data source + data views are manual steps** each fresh stack
  (Steps 4 / 4b), until they're baked into the OSD init script.
- **After restarting the Mustang node, restart Data Prepper** (`docker restart
  data-prepper`). DP caches "templates already installed" from the previous node,
  so on a fresh node it writes template-less (non-parquet) `otel-*-plain` indices
  until restarted. A full teardown+up avoids this since DP starts fresh.
