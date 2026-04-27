# EEL6761 Project — TA Demo Q&A

---

## Data flow — the big picture

The 3 stream scripts hit live biodiversity APIs (iDigBio, GBIF, OBIS) and push JSON records into Kafka topics. Each source has its own topic (`idigbio`, `gbif`, `obis`) with 3 partitions and replication factor 2. Spark reads from all 3 topics simultaneously using Structured Streaming, processes data in 2-minute micro-batch windows, and writes JSON result files to `/shared/results/` over NFS. The API server reads those JSON files to answer queries.

```
iDigBio API ──►  stream-idigbio.py ──► Kafka topic: idigbio ──┐
GBIF API    ──►  stream-gbif.py    ──► Kafka topic: gbif    ──┼──► Spark Structured Streaming
OBIS API    ──►  stream-obis.py    ──► Kafka topic: obis    ──┘         │
                                                                         ▼
                                                              /shared/results/*.json (NFS)
                                                                         │
                                                                         ▼
                                                               api_server.py :12000
```

---

## Why Kafka?

Kafka acts as a buffer between the data sources and Spark. If Spark falls behind processing, data doesn't get lost — it waits in Kafka until Spark is ready to consume it. Messages expire after 2 minutes (`retention.ms=120000`) so the cluster doesn't fill up with old data.

---

## Why KRaft (no ZooKeeper)?

Kafka 3.x supports running the controller quorum internally via KRaft mode. Each node acts as both a broker and a controller — cleaner setup with one fewer dependency. The quorum is configured with:

```
controller.quorum.voters=0@10.10.1.1:9093,1@10.10.1.2:9093,2@10.10.1.3:9093
```

---

## Why NFS?

Spark workers run on all 3 nodes but the API server only runs on node0. NFS gives every node a shared `/shared/results/` directory so Spark can write result files from any worker and the API server on node0 can read them all in one place.

- node0 exports `/shared` via `nfs-kernel-server`
- node1 and node2 mount it at `/shared`

---

## Round-robin broker assignment in `addSource`

The first `addSource` call sends stream data to `10.10.1.1:9092`, the second to `10.10.1.2:9092`, the third to `10.10.1.3:9092`. It cycles using `broker_counter[0] % 3` in `api_server.py`:

```python
broker_host = BROKER_NODES[broker_counter[0] % 3]
broker_counter[0] += 1
env["KAFKA_BROKER"] = f"{broker_host}:9092"
```

So:
| Call order | Source  | Assigned broker  |
|------------|---------|------------------|
| 1st        | any     | `10.10.1.1:9092` |
| 2nd        | any     | `10.10.1.2:9092` |
| 3rd        | any     | `10.10.1.3:9092` |

---

## What does `count?by=source` do?

Reads all JSON files in `/shared/results/`, finds the `total` field in each, and sums them grouped by `source` name. Each JSON file represents one 2-minute window from one source.

```python
for r in results:
    totals[r["source"]] = totals.get(r["source"], 0) + r["total"]
```

Example response:
```
idigbio 38420
gbif 8631
obis 19284
```

---

## What does `count?by=totalSpecies` do?

Takes the union of all `species_list` arrays across every saved window and every source, then returns the size of that set. The result is truly unique species across all sources and all time windows combined.

```python
all_species = set()
for r in results:
    all_species.update(r.get("species_list", []))
return f"species {len(all_species)}"
```

---

## What does `count?by=kingdom` do?

Reads all JSON files, aggregates the `kingdoms` dict from each window (which contains counts for `plantae`, `animalia`, `fungi`), and sums them across all windows and all sources.

```python
for r in results:
    for k, v in r.get("kingdoms", {}).items():
        totals[k] = totals.get(k, 0) + v
```

---

## How does the Spark pipeline identify kingdoms and species?

The field names differ slightly across data sources, so the pipeline uses `coalesce()` to try multiple field names in order:

```python
# Species name
col("sci_name") = coalesce(col("dwc:scientificname"), col("scientificname"), col("species"))

# Kingdom
col("kingdom_norm") = lower(coalesce(col("dwc:kingdom"), col("kingdom")))
```

It then filters for `plantae`, `animalia`, `fungi` to count kingdoms, and uses `.distinct()` on `sci_name` to count unique species per window.

---

## 400 / 404 error handling

- **404** — Any unrecognized URL path hits the `/(.*) → NotFound` catch-all route and raises `web.notfound()`.
- **400** — Invalid parameter values (wrong `name` in `addSource`/`removeSource`, or wrong `by` in `count`) raise `web.badrequest()` with a message like `"Unexpected value 'xyz'"`.

```python
# Catch-all for unknown paths
class NotFound:
    def GET(self, path):
        raise web.notfound()

# Bad parameter example
if name not in ALLOWED_SOURCES:
    raise web.badrequest(f"Unexpected value '{name}'")
```

---

## Kafka topic configuration

| Setting | Value | Reason |
|---|---|---|
| `partitions` | 3 | One per broker, enables parallel consumption |
| `replication-factor` | 2 | Data survives one broker going down |
| `retention.ms` | 120000 (2 min) | Required by spec; prevents disk fill |
| `log.retention.check.interval.ms` | 10000 | Checks for expired messages every 10s |

---

## Spark streaming configuration

| Setting | Value |
|---|---|
| Master | `spark://10.10.1.1:7077` |
| Trigger | `processingTime="2 minutes"` |
| Output mode | `append` |
| Starting offsets | `latest` |
| Package | `spark-sql-kafka-0-10_2.12:3.5.8` |

Each 2-minute micro-batch is processed by `foreachBatch`, which calls `process_batch(df, epoch_id, source)` and writes a JSON file to `/shared/results/{source}_window_{epoch_id}.json`.

---

## Node roles summary

| Node | IP | Role |
|---|---|---|
| node0 | 10.10.1.1 | Spark master, Kafka broker 0, iDigBio stream reader, API server |
| node1 | 10.10.1.2 | Spark worker, Kafka broker 1, GBIF stream reader |
| node2 | 10.10.1.3 | Spark worker, Kafka broker 2, OBIS stream reader |
