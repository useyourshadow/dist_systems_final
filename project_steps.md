# EEL6761 Project Setup Notes
# IPs: node0=10.10.1.1 | node1=10.10.1.2 | node2=10.10.1.3
# Cluster ID: F6dzFuzcQaeLo8-rX54OqA
# Hostnames: node0=pc778.emulab.net | node1=pc255.emulab.net | node2=pc774.emulab.net

---

## PHASE 1: Install Everything (ALL 3 NODES - do each separately)

```bash
sudo apt update && sudo apt install -y openjdk-11-jdk python3-pip nfs-common

pip3 install requests websocket-client kafka-python web.py pyspark

wget https://archive.apache.org/dist/spark/spark-3.5.8/spark-3.5.8-bin-hadoop3.tgz
tar -xzf spark-3.5.8-bin-hadoop3.tgz
sudo mv spark-3.5.8-bin-hadoop3 /opt/spark

wget https://downloads.apache.org/kafka/3.9.2/kafka_2.13-3.9.2.tgz
tar -xzf kafka_2.13-3.9.2.tgz
sudo mv kafka_2.13-3.9.2 /opt/kafka

echo 'export SPARK_HOME=/opt/spark' >> ~/.bashrc
echo 'export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin:/opt/kafka/bin' >> ~/.bashrc
source ~/.bashrc
```

---

## PHASE 2: Configure Kafka KRaft (EACH NODE SEPARATELY)

### node0:
```bash
sudo nano /opt/kafka/config/kraft/server.properties
```
Set these values (Ctrl+W to search for each):
```
node.id=0
listeners=PLAINTEXT://10.10.1.1:9092,CONTROLLER://10.10.1.1:9093
advertised.listeners=PLAINTEXT://10.10.1.1:9092
controller.quorum.voters=0@10.10.1.1:9093,1@10.10.1.2:9093,2@10.10.1.3:9093
controller.listener.names=CONTROLLER
log.retention.ms=120000
log.retention.check.interval.ms=10000
```
Format storage:
```bash
rm -rf /tmp/kraft-combined-logs
/opt/kafka/bin/kafka-storage.sh format -t F6dzFuzcQaeLo8-rX54OqA -c /opt/kafka/config/kraft/server.properties
```

### node1:
```bash
sudo nano /opt/kafka/config/kraft/server.properties
```
```
node.id=1
listeners=PLAINTEXT://10.10.1.2:9092,CONTROLLER://10.10.1.2:9093
advertised.listeners=PLAINTEXT://10.10.1.2:9092
controller.quorum.voters=0@10.10.1.1:9093,1@10.10.1.2:9093,2@10.10.1.3:9093
controller.listener.names=CONTROLLER
log.retention.ms=120000
log.retention.check.interval.ms=10000
```
Format storage:
```bash
rm -rf /tmp/kraft-combined-logs
/opt/kafka/bin/kafka-storage.sh format -t F6dzFuzcQaeLo8-rX54OqA -c /opt/kafka/config/kraft/server.properties
```

### node2:
```bash
sudo nano /opt/kafka/config/kraft/server.properties
```
```
node.id=2
listeners=PLAINTEXT://10.10.1.3:9092,CONTROLLER://10.10.1.3:9093
advertised.listeners=PLAINTEXT://10.10.1.3:9092
controller.quorum.voters=0@10.10.1.1:9093,1@10.10.1.2:9093,2@10.10.1.3:9093
controller.listener.names=CONTROLLER
log.retention.ms=120000
log.retention.check.interval.ms=10000
```
Format storage:
```bash
rm -rf /tmp/kraft-combined-logs
/opt/kafka/bin/kafka-storage.sh format -t F6dzFuzcQaeLo8-rX54OqA -c /opt/kafka/config/kraft/server.properties
```

---

## PHASE 3: Configure Spark (node0 only)

```bash
cp /opt/spark/conf/spark-env.sh.template /opt/spark/conf/spark-env.sh
echo 'export SPARK_MASTER_HOST=10.10.1.1' >> /opt/spark/conf/spark-env.sh
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> /opt/spark/conf/spark-env.sh

echo '10.10.1.1' > /opt/spark/conf/workers
echo '10.10.1.2' >> /opt/spark/conf/workers
echo '10.10.1.3' >> /opt/spark/conf/workers
cat /opt/spark/conf/workers  # verify all 3 IPs shown
```

Create start/stop scripts (required by project spec Task 1b):
```bash
cat > ~/start-all.sh << 'EOF'
#!/bin/bash
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties
sleep 5
/opt/spark/sbin/start-master.sh
/opt/spark/sbin/start-workers.sh spark://10.10.1.1:7077
EOF

cat > ~/stop-all.sh << 'EOF'
#!/bin/bash
/opt/spark/sbin/stop-all.sh
/opt/kafka/bin/kafka-server-stop.sh
EOF

chmod +x ~/start-all.sh ~/stop-all.sh
```

---

## PHASE 4: Configure NFS Shared Storage

### node0 only:
```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /shared/results
sudo chmod 777 /shared/results
echo '/shared *(rw,sync,no_subtree_check,no_root_squash)' | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl enable --now nfs-kernel-server
```

### node1 and node2:
```bash
sudo mkdir -p /shared
sudo mount 10.10.1.1:/shared /shared
ls /shared  # should show "results" folder
```

---

## PHASE 5: SSH Passwordless Access (node0 only)

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa.pub  # copy this output
```

### Paste the key on node1 AND node2:
```bash
echo "PASTE_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Test from node0:
```bash
ssh ojen7@10.10.1.2 "echo node1 works"
ssh ojen7@10.10.1.3 "echo node2 works"
```

---

## PHASE 6: Start Everything (EXACT ORDER)

### Step 1 - Start Kafka on ALL 3 NODES (node0 first, then 1, then 2):
```bash
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties
```
Verify all 3 brokers visible from node0 (wait 10 seconds first):
```bash
sleep 10
/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server 10.10.1.1:9092 | grep "id:"
# Must show 3 broker IDs before continuing
```

### Step 2 - Create Kafka topics (node0 only):
```bash
/opt/kafka/bin/kafka-topics.sh --create --topic idigbio --bootstrap-server 10.10.1.1:9092 --partitions 3 --replication-factor 2 --config retention.ms=120000
/opt/kafka/bin/kafka-topics.sh --create --topic gbif    --bootstrap-server 10.10.1.1:9092 --partitions 3 --replication-factor 2 --config retention.ms=120000
/opt/kafka/bin/kafka-topics.sh --create --topic obis    --bootstrap-server 10.10.1.1:9092 --partitions 3 --replication-factor 2 --config retention.ms=120000

/opt/kafka/bin/kafka-topics.sh --list --bootstrap-server 10.10.1.1:9092
# Should show: gbif, idigbio, obis
```

### Step 3 - Start Spark (node0 only):
```bash
/opt/spark/sbin/start-master.sh
/opt/spark/sbin/start-workers.sh spark://10.10.1.1:7077
# Check UI at: http://pc778.emulab.net:8080 (should show 3 workers)
```

### Step 4 - Submit Spark pipeline (node0 terminal 1):
```bash
/opt/spark/bin/spark-submit \
  --master spark://10.10.1.1:7077 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.8 \
  ~/spark_pipeline.py
# WAIT until you see "Streaming query started" before starting stream scripts
```

### Step 5 - Start data streams (3 separate terminals simultaneously):
node0 terminal 2:
```bash
python3 ~/stream-idigbio.py
```
node1 shell:
```bash
python3 ~/stream-gbif.py
```
node2 shell:
```bash
python3 ~/stream-obis.py
```

### Step 6 - Start API server (node0 terminal 3):
```bash
# First make sure all 3 stream scripts exist on node0
scp ojen7@10.10.1.2:~/stream-gbif.py ~/
scp ojen7@10.10.1.3:~/stream-obis.py ~/
ls ~/stream-idigbio.py ~/stream-gbif.py ~/stream-obis.py  # verify

python3 ~/api_server.py 12000
```

---

## PHASE 7: Test the API (node0, any terminal)

```bash
# Normal requests (Task 2b, 2c, 2d)
curl "http://localhost:12000/addSource?name=idigbio"
curl "http://localhost:12000/addSource?name=gbif"
curl "http://localhost:12000/addSource?name=obis"
curl "http://localhost:12000/listSources"
curl "http://localhost:12000/count?by=kingdom"
curl "http://localhost:12000/count?by=source"
curl "http://localhost:12000/count?by=totalSpecies"

# Extra credit - remove source (Task 2f)
curl "http://localhost:12000/removeSource?name=idigbio"
curl "http://localhost:12000/listSources"  # should no longer show idigbio

# Error responses (Task 2e - must return correct codes)
curl -v "http://localhost:12000/count?by=invalid"      # must return 400
curl -v "http://localhost:12000/badpath"                # must return 404
curl -v "http://localhost:12000/addSource?name=bogus"   # must return 400
curl -v "http://localhost:12000/removeSource?name=xyz"  # must return 400
```

---

## PHASE 8: Collect Results (after 20 minutes / 10 windows)

```bash
ls /shared/results/   # should show 30 json files (3 sources x 10 windows)
python3 ~/generate_report.py
```

---

## ALL SCRIPT CONTENTS

### ~/spark_pipeline.py
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import json, os

spark = SparkSession.builder.appName("BiodiversityPipeline").getOrCreate()
spark.sparkContext.setLogLevel("WARN")
os.makedirs("/shared/results", exist_ok=True)

schema = StructType([
    StructField("kingdom", StringType(), True),
    StructField("scientificname", StringType(), True),
    StructField("species", StringType(), True),
])

brokers = "10.10.1.1:9092,10.10.1.2:9092,10.10.1.3:9092"

def process_batch(df, epoch_id, source):
    if df.count() == 0:
        return
    df = df.withColumn(
        "sci_name", coalesce(col("scientificname"), col("species"))
    ).withColumn("kingdom_norm", lower(col("kingdom")))
    kingdom_counts = df.filter(
        col("kingdom_norm").isin(["plantae", "animalia", "fungi"])
    ).groupBy("kingdom_norm").count().collect()
    unique_species = df.select("sci_name").distinct().count()
    total = df.count()
    species_list = [r["sci_name"] for r in df.select("sci_name").distinct().collect() if r["sci_name"]]
    result = {
        "window": epoch_id, "source": source, "total": total,
        "unique_species": unique_species, "species_list": species_list,
        "kingdoms": {r["kingdom_norm"]: r["count"] for r in kingdom_counts}
    }
    with open(f"/shared/results/{source}_window_{epoch_id}.json", "w") as f:
        json.dump(result, f)
    print(f"Saved window {epoch_id} for {source}")

queries = []
for source in ["idigbio", "gbif", "obis"]:
    df = spark.readStream.format("kafka") \
        .option("kafka.bootstrap.servers", brokers) \
        .option("subscribe", source) \
        .option("startingOffsets", "latest").load()
    parsed = df.select(from_json(col("value").cast("string"), schema).alias("data")).select("data.*")
    query = parsed.writeStream.outputMode("append") \
        .trigger(processingTime="2 minutes") \
        .foreachBatch(lambda df, epoch, s=source: process_batch(df, epoch, s)).start()
    queries.append(query)

spark.streams.awaitAnyTermination()
```

### ~/api_server.py
```python
import web, subprocess, os, json, glob

urls = (
    '/addSource',    'AddSource',
    '/removeSource', 'RemoveSource',
    '/listSources',  'ListSources',
    '/count',        'Count',
    '/(.*)',         'NotFound'
)
app = web.application(urls, globals())

ALLOWED_SOURCES = ["idigbio", "gbif", "obis"]
active_sources = {}
BROKER_NODES = {0: "10.10.1.1", 1: "10.10.1.2", 2: "10.10.1.3"}
broker_counter = [0]
script_map = {
    "idigbio": "/users/ojen7/stream-idigbio.py",
    "gbif":    "/users/ojen7/stream-gbif.py",
    "obis":    "/users/ojen7/stream-obis.py"
}

class AddSource:
    def GET(self):
        params = web.input(name=None)
        name = params.name
        if name not in ALLOWED_SOURCES:
            raise web.badrequest(f"Unexpected value '{name}'")
        if name not in active_sources:
            broker_host = BROKER_NODES[broker_counter[0] % 3]
            broker_counter[0] += 1
            env = os.environ.copy()
            env["KAFKA_BROKER"] = f"{broker_host}:9092"
            active_sources[name] = subprocess.Popen(["python3", script_map[name]], env=env)
        return ""

class RemoveSource:
    def GET(self):
        params = web.input(name=None)
        name = params.name
        if name not in ALLOWED_SOURCES:
            raise web.badrequest(f"Unexpected value '{name}'")
        if name in active_sources:
            active_sources[name].terminate()
            del active_sources[name]
        return ""

class ListSources:
    def GET(self):
        web.header("Content-Type", "text/plain")
        return "\n".join(active_sources.keys())

class Count:
    def GET(self):
        params = web.input(by=None)
        by = params.by
        if by not in ["source", "kingdom", "totalSpecies"]:
            raise web.badrequest(f"Unexpected value '{by}'")
        results = [json.load(open(f)) for f in glob.glob("/shared/results/*.json")]
        web.header("Content-Type", "text/plain")
        if by == "source":
            totals = {}
            for r in results:
                totals[r["source"]] = totals.get(r["source"], 0) + r["total"]
            return "\n".join(f"{k} {v}" for k, v in totals.items())
        elif by == "kingdom":
            totals = {}
            for r in results:
                for k, v in r.get("kingdoms", {}).items():
                    totals[k] = totals.get(k, 0) + v
            return "\n".join(f"{k} {v}" for k, v in totals.items())
        elif by == "totalSpecies":
            all_species = set()
            for r in results:
                all_species.update(r.get("species_list", []))
            return f"species {len(all_species)}"

class NotFound:
    def GET(self, path):
        raise web.notfound()

if __name__ == "__main__":
    app.run()
```

### ~/generate_report.py
```python
import json, glob

for source in ["idigbio", "gbif", "obis"]:
    print(f"\n=== {source.upper()} ===")
    print(f"{'Window':<8} {'Plants':<10} {'Animals':<10} {'Fungi':<10} {'Unique Species'}")
    print("-" * 55)
    files = sorted(glob.glob(f"/shared/results/{source}_window_*.json"),
                   key=lambda x: int(x.split("_window_")[1].replace(".json","")))
    for i, f in enumerate(files, 1):
        r = json.load(open(f))
        k = r.get("kingdoms", {})
        print(f"{i:<8} {k.get('plantae',0):<10} {k.get('animalia',0):<10} {k.get('fungi',0):<10} {r.get('unique_species',0)}")
```

---

## STOP/RESTART COMMANDS

```bash
# Kill stream scripts
pkill -f stream-idigbio.py
pkill -f stream-gbif.py
pkill -f stream-obis.py

# Stop Spark (node0)
/opt/spark/sbin/stop-all.sh

# Stop Kafka (run on ALL 3 nodes)
/opt/kafka/bin/kafka-server-stop.sh

# Delete and recreate topics if needed (node0)
/opt/kafka/bin/kafka-topics.sh --bootstrap-server 10.10.1.1:9092 --delete --topic idigbio
/opt/kafka/bin/kafka-topics.sh --bootstrap-server 10.10.1.1:9092 --delete --topic gbif
/opt/kafka/bin/kafka-topics.sh --bootstrap-server 10.10.1.1:9092 --delete --topic obis
```

---

## SUBMISSION CHECKLIST (from project PDF)

- [ ] 10 windows of Spark output tables in PDF (one table per data source)
- [ ] CPU utilization vs time plot for each node in PDF
- [ ] Brief description of setup and scripts in PDF
- [ ] Full source code of all scripts in PDF
- [ ] API tested with curl including malformed/bad requests
- [ ] Live demo ready: explain data readers → Kafka → Spark → API

## CURRENT STATUS
- [x] Java installed on all nodes
- [x] Spark installed on all nodes
- [x] Kafka installed on all nodes
- [x] Python deps installed on all nodes
- [x] Spark cluster configured (workers file, spark-env.sh)
- [x] NFS set up
- [x] SSH passwordless working
- [ ] Kafka KRaft 3-broker cluster working  <-- CURRENTLY FIXING
- [ ] Spark pipeline running 10 windows
- [ ] API server tested
- [ ] generate_report.py producing tables
