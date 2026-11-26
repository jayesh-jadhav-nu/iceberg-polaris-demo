# ShopCo lakehouse demo: Apache Iceberg + Apache Polaris

A complete, runnable demonstration of Apache Iceberg and Apache Polaris on one laptop, built around a single use case: ShopCo keeps one copy of its sales data in object storage, and four teams on four engines (a streaming writer, Spark, Trino, PyIceberg) share it safely through Polaris.

Verified end to end on Linux and macOS with `apache/polaris` 1.9.0, Spark 3.5 with Iceberg 1.8.1, Trino and RustFS 1.0.1.

## Repository layout

| Path | What it is |
|---|---|
| `shopco-demo-kit.tar.gz` | The demo kit: `run.sh` (one command per step), Polaris setup script, one script per step (5 to 16), `DEMO_GUIDE.md` (expected output, theory and likely questions for every step), a Postman collection for the Polaris APIs, and the log of the last clean run |
| `decks/ShopCo_Iceberg_Polaris_Walkthrough.pptx` | The deck, 32 slides. The notes pane of every slide explains what the slide shows and why |
| `decks/ShopCo_Iceberg_Polaris_Walkthrough.pdf` | The slides as a PDF |
| `ShopCo_Lakehouse_Walkthrough.pdf` | The written walkthrough: the design end to end, with a *Demo* line under every step and the commands behind each |
| `README.md` | This file |


## What you need

- Docker Desktop, running, with at least **10 GB of memory** (Settings → Resources → Memory). Images take about 13 GB of disk on first pull.
- `curl` and `jq` (`brew install jq` on macOS).
- A terminal. Every command below runs from the `demo-kit` folder once it is unpacked.

## 1. Unpack the kit

From the repository root:

```bash
tar xzf shopco-demo-kit.tar.gz
cd demo-kit
chmod +x run.sh polaris-setup.sh demo/*.sh
./run.sh            # with no argument it prints the list of steps
```

## 2. Set up and run through step 7

Steps 1 to 6 are the setup; they are normally done before an audience arrives. Step 7 is the first live step. Run them one at a time, or paste the whole block:

```bash
./run.sh up        # steps 1-2: start RustFS (the S3 stand-in), create the shopco-lake bucket, start Polaris
./run.sh setup     # steps 3-4: create the retail catalog, 5 catalog roles, grants, 5 principal roles, 6 principals
./run.sh engines   # start Spark and Trino, each logged in to Polaris as its own principal
./run.sh 5         # create the raw, curated and pii namespaces and the three tables; seed 50 customers
./run.sh grants    # namespace- and table-level grants (they need the tables to exist)
./run.sh 6         # start the streaming writer: a batch of orders into raw.orders every 30 seconds
```

Give the stream about two minutes, and confirm it is alive:

```bash
./run.sh status    # five containers, "stream: RUNNING"
./run.sh snapshots # append snapshots 30 seconds apart
```

Then the first live step:

```bash
./run.sh 7         # the nightly job: write to the audit branch, check it, fast-forward main; the stream commits meanwhile
./run.sh 7b        # incremental read: only the rows added since the last run, bookmarked with a tag
```

Each step prints the same way: a banner naming the capabilities, BEFORE, ACTION, AFTER, and a WHAT CHANGED line. `DEMO_GUIDE.md` in the kit shows the expected output of every step and what it means.

## 3. Continue after step 7

Run the remaining steps in order. Each is independent enough to pause between them; the stream keeps running until step 12 stops it.

```bash
./run.sh 8         # change the live table: add and rename a column, widen a type, daily -> hourly partitions; shows the change in metadata.json and in the Parquet files
./run.sh 9         # query as the analyst in Trino: two statements allowed, four refused for different reasons
./run.sh 10        # pin the training data with a tag; read it from Python (PyIceberg) as the data scientist
./run.sh 11        # a bad UPDATE, then rollback
./run.sh 12        # forget a customer: the logical delete (stops the stream first)
./run.sh 13        # maintenance: compaction, delete-file cleanup, snapshot expiry; the physical purge
./run.sh 14        # credential vending on the wire: an allowed loadTable with temporary keys, a refused one with a 403
./run.sh 15        # the files in the bucket (or browse http://localhost:9001, admin / password)
./run.sh 16        # two writers on the same table: Polaris refuses the stale commit, the writer retries
```

Useful at any point:

```bash
./run.sh inspect   # everything Polaris knows: catalog, roles and grants, principals, namespaces, table schemas
./run.sh files     # the data files Iceberg currently counts as part of raw.orders, to compare with the bucket
./run.sh sql       # interactive Spark SQL shell (as spark_etl)
./run.sh trino     # interactive Trino shell (as trino_bi)
./run.sh stop-stream
```

Steps 8, 11, 12, 13 and 16 are safe to rerun. Step 7 can be rerun as well (it updates the same five rows). If the stream has been stopped, `./run.sh 6` restarts it from its checkpoint.

## 4. Reset

Removes the containers, volumes, secrets and generated configs. Start again from section 2 afterwards.

```bash
./run.sh reset
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `./run.sh up` never says "Polaris is ready" | First image pull, or Docker has too little memory | Wait; check `docker compose logs polaris`; raise Docker's memory to 10 GB |
| "Forbidden" or 401 from Spark | Spark started before `setup` wrote its config | `docker compose restart spark`, wait a minute |
| Trino says catalog `retail` does not exist | Trino started before `setup` wrote its config | `./run.sh setup` then `docker compose restart trino` |
| `./run.sh snapshots` shows nothing new | The stream died or step 12 stopped it | `./run.sh status` prints its last log lines; `./run.sh 6` restarts it |
| Docker says no space left | The images are about 13 GB | Free disk or raise Docker's disk limit |

More cases, with the exact messages, are in `demo-kit/DEMO_GUIDE.md`.
