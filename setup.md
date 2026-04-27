
# EEL6761 Project — Demo Command Reference

**Cluster:** node0 `10.10.1.1` · node1 `10.10.1.2` · node2 `10.10.1.3`  
**Hostnames:** node0 `pc778.emulab.net` · node1 `pc255.emulab.net` · node2 `pc774.emulab.net`  
**Cluster ID:** `F6dzFuzcQaeLo8-rX54OqA`

---

## Phase 0 — Stop Everything First

### node0
```bash
pkill -f stream-idigbio.py
pkill -f stream-gbif.py
pkill -f stream-obis.py
pkill -f api_server.py
/opt/spark/sbin/stop-all.sh
/opt/kafka/bin/kafka-server-stop.sh
```

### node1 and node2
```bash
/opt/kafka/bin/kafka-server-stop.sh
```

### node0 — clear old results (optional but recommended)
```bash
rm -f /shared/results/*.json
```
> Prevents stale data from appearing in API responses during the demo.

---

## Phase 1 — Start Kafka

Run on **node0 first**, then node1, then node2:
```bash
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties
```

### node0 — verify all 3 brokers are up (wait ~10s first)
```bash
sleep 10 && /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server 10.10.1.1:9092 | grep "id:"
```
> **Must see 3 broker IDs before continuing.** If not, Kafka isn't fully up yet.

### node0 — recreate topics
```bash
for t in idigbio gbif obis; do
  /opt/kafka/bin/kafka-topics.sh --delete --bootstrap-server 10.10.1.1:9092 --topic $t 2>/dev/null
  /opt/kafka/bin/kafka-topics.sh --create --bootstrap-server 10.10.1.1:9092 --topic $t \
    --partitions 3 --replication-factor 2 --config retention.ms=120000
done

/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server 10.10.1.1:9092
```
> Should list: `gbif`, `idigbio`, `obis`

---

## Phase 2 — Start Spark

### node0
```bash
/opt/spark/sbin/start-master.sh
/opt/spark/sbin/start-workers.sh spark://10.10.1.1:7077
```
> Verify at **http://pc778.emulab.net:8080** — must show 3 workers alive.

---

## Phase 3 — Submit Spark Pipeline

### node0 — terminal 1
```bash
/opt/spark/bin/spark-submit \
  --master spark://10.10.1.1:7077 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.8 \
  ~/spark_pipeline.py
```
> **Wait until you see `"Streaming query started"` before starting the stream scripts.**

---

## Phase 4 — Start Streams + API Server

Open separate terminals for each. Start all four roughly simultaneously.

### node0 — terminal 2
```bash
python3 ~/stream-idigbio.py
```

### node1
```bash
python3 ~/stream-gbif.py
```

### node2
```bash
python3 ~/stream-obis.py
```

### node0 — terminal 3
```bash
python3 ~/api_server.py 12000
```

---

## Phase 5 — API Demo Calls

Run from any free terminal on node0.

### Normal calls
```bash
curl "http://localhost:12000/addSource?name=idigbio"
curl "http://localhost:12000/addSource?name=gbif"
curl "http://localhost:12000/addSource?name=obis"
curl "http://localhost:12000/listSources"
curl "http://localhost:12000/count?by=kingdom"
curl "http://localhost:12000/count?by=source"
curl "http://localhost:12000/count?by=totalSpecies"
```

### Extra credit — removeSource
```bash
curl "http://localhost:12000/removeSource?name=idigbio"
curl "http://localhost:12000/listSources"   # idigbio should no longer appear
```

### Error handling — must return correct HTTP codes
```bash
curl -v "http://localhost:12000/count?by=invalid"       # expect 400
curl -v "http://localhost:12000/badpath"                # expect 404
curl -v "http://localhost:12000/addSource?name=bogus"   # expect 400
curl -v "http://localhost:12000/removeSource?name=xyz"  # expect 400
```

---

## Troubleshooting — Kafka Won't Start

Only run this if Kafka fails with a storage formatting error. Run on **all 3 nodes**, then repeat Phase 1.

```bash
rm -rf /tmp/kraft-combined-logs
/opt/kafka/bin/kafka-storage.sh format \
  -t F6dzFuzcQaeLo8-rX54OqA \
  -c /opt/kafka/config/kraft/server.properties
```

---

## Stop / Restart Commands

```bash
# Kill stream scripts (node0)
pkill -f stream-idigbio.py
pkill -f stream-gbif.py
pkill -f stream-obis.py

# Stop Spark (node0)
/opt/spark/sbin/stop-all.sh

# Stop Kafka (run on ALL 3 nodes)
/opt/kafka/bin/kafka-server-stop.sh
```

---

## Quick Reference — API Endpoints

| Endpoint | Response | Notes |
|---|---|---|
| `GET /addSource?name=[NAME]` | `200` / `400` | Allowed: `idigbio`, `gbif`, `obis` |
| `GET /removeSource?name=[NAME]` | `200` / `400` | Stops the stream subprocess |
| `GET /listSources` | `200` | Newline-separated active sources |
| `GET /count?by=source` | `200` / `400` | Total records per source |
| `GET /count?by=kingdom` | `200` / `400` | Records by kingdom |
| `GET /count?by=totalSpecies` | `200` / `400` | Unique species count |
| `GET /[anything else]` | `404` | Catch-all not found |
