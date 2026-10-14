# Lab 00 — Environment Setup

> **Goal:** Build, on your own laptop, the complete local data platform that every SK-03 lab and the end-to-end guided lab run on: MinIO (S3), Spark, Airflow, a Postgres warehouse, and dbt.
> **Time:** 180–300 min (mostly downloads) | **Difficulty:** Beginner | **Prerequisites:** SK-01 and SK-02 completed
> **Guided lab coverage:** PREREQUISITES (the whole local stack); the local equivalents of AWS S3, Glue, Redshift and Slack used throughout Parts 1–4

## Learning objectives

By the end of this lab you can:

- Explain what Docker, an image, a container, and Docker Compose are — and why data platforms ship as containers.
- Start, stop, inspect, and reset an eight-container platform with `docker compose`.
- Say which hostname to use from Windows (`localhost:<port>`) and which to use from inside a container (the service name, e.g. `minio:9000`).
- Map each local service to the AWS service it stands in for in the guided lab.

| Guided lab (AWS) | This lab (local, free) | How you reach it |
|---|---|---|
| Amazon S3 bucket `urbangear-data-lake` | **MinIO** container `minio` | Console http://localhost:9001 · code: `http://minio:9000` |
| AWS Glue (Spark) | **PySpark** inside the Airflow image (+ a small Spark cluster for learning) | `spark-submit` · Spark UI http://localhost:8081 |
| Airflow (MWAA) | **Airflow 2.9.3** containers | http://localhost:8080 |
| Amazon Redshift | **Postgres 16** container `warehouse` | Windows: `localhost:5433` · code: `warehouse:5432` |
| dbt Cloud / dbt in CI | **dbt-postgres 1.8** inside the Airflow image | `docker exec ... dbt ...` |
| Slack alerts | A logged `[ALERT]` line (real Slack optional) | Airflow task logs |

## Steps

### Step 1 — Check your machine

| Requirement | Minimum | How to check |
|---|---|---|
| Windows 11 (or Windows 10 21H2+) | 64-bit, virtualization enabled | Task Manager → Performance → CPU → "Virtualization: Enabled" |
| RAM | 16 GB recommended (8 GB works, tightly) | Task Manager → Performance |
| Free disk space | **25 GB** on C: | File Explorer → This PC |
| Admin rights | Needed for installs | — |

If virtualization is **Disabled**, reboot into BIOS/UEFI (F2, F10 or Del at boot) and enable "Intel VT-x" / "AMD-V" / "SVM Mode".

**Expected output:** all four rows satisfied.

✅ **Checkpoint:** virtualization enabled and ≥ 25 GB free.

### Step 2 — Install WSL2 and Docker Desktop

**WSL2** (Windows Subsystem for Linux 2) is a small Linux virtual machine built into Windows. Containers are a Linux technology, so Docker Desktop runs them inside WSL2.

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

Restart when asked. Then install **Docker Desktop** from https://www.docker.com/products/docker-desktop/ (keep "Use WSL 2 instead of Hyper-V" ticked), start it, and wait until it says **Engine running**.

Cap the memory Docker may use: create `C:\Users\<YourName>\.wslconfig` containing

```ini
[wsl2]
memory=10GB
processors=4
```

(use `memory=6GB` on an 8 GB laptop), then run `wsl --shutdown` and restart Docker Desktop.

Verify in a **normal** PowerShell window:

```powershell
wsl --status
docker --version
docker run --rm hello-world
```

**Expected output:**

```
Default Version: 2
Docker version 27.x.x, build ...
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

**What just happened:** Docker downloaded a tiny **image** (a packaged filesystem + start command), ran it as a **container** (a running instance of the image), printed a message, and deleted the container (`--rm`). *Image = recipe; container = the cake.*

✅ **Checkpoint:** you saw `Hello from Docker!`.

### Step 3 — Create the project folder

All SK-03 work lives in **one** folder. Keep it **out of OneDrive** (and Dropbox/Google Drive): sync tools lock and rewrite files while Docker is reading them, which causes baffling errors.

```powershell
mkdir C:\DataEngineering\sk03
cd C:\DataEngineering\sk03
mkdir dags, jobs, dbt, dbt\profiles, data, logs, plugins, docker, sql
dir
```

**Expected output:** a listing with these nine folders: `dags`, `data`, `dbt`, `docker`, `jobs`, `logs`, `plugins`, `sql` (and `dbt\profiles` inside `dbt`).

| Folder | What goes in it | First used |
|---|---|---|
| `jobs\` | Python and PySpark scripts (`generate_orders.py`, `etl_job.py`, ...) | Lab 01 |
| `dags\` | Airflow DAG files | Lab 05 |
| `dbt\urbangear\`, `dbt\profiles\` | The dbt project and its connection profile | Lab 08 |
| `sql\` | `warehouse_init.sql` — creates the warehouse schemas/table | this lab |
| `docker\` | The custom Airflow `Dockerfile` | this lab |
| `logs\`, `plugins\`, `data\` | Airflow logs, (unused) plugins, scratch files | — |

✅ **Checkpoint:** `cd C:\DataEngineering\sk03` works and `dir` shows the folders.

### Step 4 — Write the custom Airflow image (`docker\Dockerfile`)

The stock Airflow image has no Java, PySpark or dbt. We extend it. Create `C:\DataEngineering\sk03\docker\Dockerfile` (VS Code: `code .` from the project folder):

```dockerfile
FROM apache/airflow:2.9.3-python3.11

# ---- Java: PySpark runs on the JVM ---------------------------------------
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
# TARGETARCH is filled in by Docker (amd64 on most PCs, arm64 on Apple Silicon)
ARG TARGETARCH
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-${TARGETARCH}

# ---- Python packages (always as the airflow user, never as root) ---------
# Re-stating apache-airflow==2.9.3 stops pip from silently upgrading Airflow
# while it resolves the new packages' dependencies.
USER airflow
RUN pip install --no-cache-dir \
    "apache-airflow==2.9.3" \
    "pyspark==3.5.1" \
    "dbt-core~=1.8.0" \
    "dbt-postgres~=1.8.0"

# ---- S3A connector JARs, baked into PySpark's own jars/ folder ------------
# Spark needs these two JARs to read/write s3a:// paths (MinIO or AWS S3).
# Baking them in means no download at job start-up and no internet at run time.
RUN JARS="$(python -c 'import os, pyspark; print(os.path.join(os.path.dirname(pyspark.__file__), "jars"))')" \
    && curl -fsSL -o "$JARS/hadoop-aws-3.3.4.jar" \
       https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.3.4/hadoop-aws-3.3.4.jar \
    && curl -fsSL -o "$JARS/aws-java-sdk-bundle-1.12.262.jar" \
       https://repo1.maven.org/maven2/com/amazonaws/aws-java-sdk-bundle/1.12.262/aws-java-sdk-bundle-1.12.262.jar
```

Why each part matters:

- **Java 17** — Spark runs on the Java Virtual Machine; PySpark is a Python remote control for it.
- **Pinned versions** (`2.9.3`, `3.5.1`, `~=1.8.0`) — the same versions as the guided lab. Unpinned installs are how "it worked yesterday" happens.
- **The two S3A JARs** — teach Spark to read/write `s3a://` paths. Baking them into the image means no download when a job starts.

**Expected output:** none yet (the file is built in Step 7).

✅ **Checkpoint:** `C:\DataEngineering\sk03\docker\Dockerfile` exists.

### Step 5 — Write the warehouse bootstrap SQL (`sql\warehouse_init.sql`)

Postgres runs every `.sql` file in `/docker-entrypoint-initdb.d/` **once**, when its data volume is first created. Ours creates the schemas, the staging table the pipeline loads into (Lab 07), and the read-only group dbt grants access to (Lab 08). Create `C:\DataEngineering\sk03\sql\warehouse_init.sql`:

```sql
-- SK-03 warehouse bootstrap. Runs ONCE, when the warehouse volume is first created.
-- (Changed this file? Re-run it with:  docker exec -i warehouse psql -U dwh_user -d warehouse < sql\warehouse_init.sql)

CREATE SCHEMA IF NOT EXISTS staging;     -- raw loads from the data lake (pipeline writes here)
CREATE SCHEMA IF NOT EXISTS analytics;   -- dbt marts (analysts read here)

-- Landing table for processed orders. One row per order line.
-- Columns = the Spark job's clean output (Lab 04 / guided lab Step 1.4).
CREATE TABLE IF NOT EXISTS staging.daily_orders (
    order_id            TEXT,
    customer_id         TEXT,
    platform            TEXT,
    order_date          TIMESTAMP,
    product_id          TEXT,
    product_name        TEXT,
    category            TEXT,
    quantity            INTEGER,
    unit_price          DOUBLE PRECISION,
    discount_pct        DOUBLE PRECISION,
    shipping_cost       DOUBLE PRECISION,
    tax_amount          DOUBLE PRECISION,
    payment_method      TEXT,
    shipping_country    TEXT,
    shipping_state      TEXT,
    shipping_city       TEXT,
    is_returned         BOOLEAN,
    gross_revenue       DOUBLE PRECISION,
    discount_amount     DOUBLE PRECISION,
    net_revenue         DOUBLE PRECISION,
    total_amount        DOUBLE PRECISION,
    order_year          INTEGER,
    order_month         INTEGER,
    order_day           INTEGER,
    order_hour          INTEGER,
    day_of_week         INTEGER,
    is_weekend          BOOLEAN,
    order_item_count    BIGINT,
    order_total_revenue DOUBLE PRECISION,
    is_high_value       BOOLEAN,
    etl_processed_at    TIMESTAMP,
    etl_process_date    DATE
);

-- Read-only group for analysts; dbt's post_hook GRANTs marts to it.
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'analytics_readers') THEN
        CREATE ROLE analytics_readers NOLOGIN;
    END IF;
END $$;
GRANT USAGE ON SCHEMA analytics TO analytics_readers;
```

**Expected output:** none yet.

✅ **Checkpoint:** the file exists and ends with the `GRANT USAGE ...` line.

### Step 6 — Write the platform (`docker-compose.yml`)

**Docker Compose** declares many containers — their images, ports, folders and passwords — in one YAML file and starts them with one command. This file **is** your data platform. Create `C:\DataEngineering\sk03\docker-compose.yml`:

```yaml
# SK-03 local batch data platform
# Start:  docker compose up -d
# Stop:   docker compose stop          (keeps everything)
# Remove: docker compose down          (keeps data volumes)
# Reset:  docker compose down -v       (DELETES all data volumes)

x-airflow-common: &airflow-common
  build: ./docker
  image: sk03-airflow:2.9.3
  environment: &airflow-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db/airflow
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__CORE__DEFAULT_TIMEZONE: utc
    # Encrypts passwords stored in Airflow Connections. LOCAL LAB VALUE ONLY —
    # in production this comes from a secrets manager, never a file in git.
    AIRFLOW__CORE__FERNET_KEY: "iXv35_nnQa1h2LIZoCvO_QmRtEMiPgBQmE1_DBcprSE="
    AIRFLOW__WEBSERVER__SECRET_KEY: "sk03-local-lab-only"
    # MinIO credentials (S3 stand-in) — local lab values only
    MINIO_ENDPOINT: http://minio:9000
    MINIO_ACCESS_KEY: minioadmin
    MINIO_SECRET_KEY: minioadmin123
    # Analytics warehouse (Postgres) — used by dbt and the load step
    WAREHOUSE_HOST: warehouse
    WAREHOUSE_PORT: "5432"
    WAREHOUSE_DB: warehouse
    WAREHOUSE_USER: dwh_user
    WAREHOUSE_PASSWORD: dwh_pass
  volumes:
    - ./dags:/opt/airflow/dags
    - ./jobs:/opt/airflow/jobs
    - ./dbt:/opt/airflow/dbt
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
  depends_on:
    airflow-db:
      condition: service_healthy

services:

  # MinIO — S3-compatible object storage (local stand-in for Amazon S3).
  # API on 9000 (what code talks to), web console on 9001 (what YOU use).
  # Image note: MinIO no longer publishes images on Docker Hub, and Bitnami moved
  # its free images to "bitnamilegacy/*" (frozen: no more updates or patches).
  # A frozen, pinned image is fine for a local lab — never for production.
  minio:
    image: bitnamilegacy/minio:2025.4.22
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - minio-data:/bitnami/minio/data

  # Airflow's own metadata database (DAG runs, task states, connections).
  airflow-db:
    image: postgres:16
    container_name: airflow-db
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - airflow-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U airflow"]
      interval: 10s
      timeout: 5s
      retries: 5

  # The analytics WAREHOUSE (local stand-in for Amazon Redshift).
  # Host port 5433 so it never clashes with a Postgres on 5432 (e.g. SK-02's).
  # sql/warehouse_init.sql runs once, the first time the volume is created.
  warehouse:
    image: postgres:16
    container_name: warehouse
    environment:
      POSTGRES_USER: dwh_user
      POSTGRES_PASSWORD: dwh_pass
      POSTGRES_DB: warehouse
    ports:
      - "5433:5432"
    volumes:
      - warehouse-data:/var/lib/postgresql/data
      - ./sql/warehouse_init.sql:/docker-entrypoint-initdb.d/01_warehouse_init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dwh_user -d warehouse"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Spark standalone cluster (master + 1 worker) — used to learn the cluster
  # model and the Spark UI. Pipeline jobs run in local mode inside Airflow.
  spark-master:
    image: bitnamilegacy/spark:3.5
    container_name: spark-master
    environment:
      SPARK_MODE: master
    ports:
      - "8081:8080"   # Spark master UI (8081 on your PC; Airflow owns 8080)
      - "7077:7077"
    volumes:
      - ./jobs:/opt/jobs

  spark-worker:
    image: bitnamilegacy/spark:3.5
    container_name: spark-worker
    environment:
      SPARK_MODE: worker
      SPARK_MASTER_URL: spark://spark-master:7077
      SPARK_WORKER_MEMORY: 2G
      SPARK_WORKER_CORES: "2"
    depends_on:
      - spark-master
    volumes:
      - ./jobs:/opt/jobs

  # Airflow: one-time init, then webserver (UI on 8080) and scheduler.
  airflow-init:
    <<: *airflow-common
    container_name: airflow-init
    entrypoint: /bin/bash
    command:
      - -c
      - |
        airflow db migrate
        airflow users create \
          --username admin --password admin \
          --firstname Admin --lastname User \
          --role Admin --email admin@example.com || true
    restart: "no"

  airflow-webserver:
    <<: *airflow-common
    container_name: airflow-webserver
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 5
    depends_on:
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    container_name: airflow-scheduler
    command: scheduler
    depends_on:
      airflow-init:
        condition: service_completed_successfully

volumes:
  minio-data:
  airflow-db-data:
  warehouse-data:
```

Read these four ideas carefully — they come up in every lab:

- **Service names are hostnames.** Inside the Docker network, containers reach each other by service name: code in Airflow uses `http://minio:9000` and host `warehouse`. From **Windows** you use `localhost` and the *published* port (left side of `"5433:5432"`). Mixing these up is the #1 beginner error.
- **Bind mounts** (`./jobs:/opt/airflow/jobs`) make a Windows folder appear inside the container. Save `jobs\etl_job.py` in VS Code → the container sees it immediately.
- **Named volumes** (`minio-data:`) keep buckets and databases across restarts. `docker compose down -v` deletes them.
- **Anchors** (`&airflow-common` / `<<: *airflow-common`) define the Airflow settings once and reuse them for init, webserver and scheduler.

> **About the `bitnamilegacy/*` images.** MinIO stopped publishing its own images on Docker Hub, and Bitnami moved its free images to the `bitnamilegacy` namespace in 2025. Those images are **frozen**: they still download, but they receive no updates or security patches. We pin `bitnamilegacy/minio:2025.4.22` and `bitnamilegacy/spark:3.5` because they exist, need no extra configuration, and keep this lab reproducible. That is acceptable for a throw-away local lab; in production you would use a maintained image (or the real AWS service).

**Expected output:** none yet.

✅ **Checkpoint:** `C:\DataEngineering\sk03` contains `docker-compose.yml`, `docker\Dockerfile`, `sql\warehouse_init.sql`.

### Step 7 — Build and start everything

```powershell
cd C:\DataEngineering\sk03
docker compose up -d --build
```

The first run builds the Airflow image and downloads the others (several GB, 10–30 minutes). Later starts take seconds.

```powershell
docker compose ps -a
```

**Expected output** (columns trimmed; your "CREATED/STATUS" times differ):

```
NAME                IMAGE                           SERVICE             STATUS
airflow-db          postgres:16                     airflow-db          Up (healthy)
airflow-init        sk03-airflow:2.9.3              airflow-init        Exited (0)
airflow-scheduler   sk03-airflow:2.9.3              airflow-scheduler   Up
airflow-webserver   sk03-airflow:2.9.3              airflow-webserver   Up (healthy)
minio               bitnamilegacy/minio:2025.4.22   minio               Up
spark-master        bitnamilegacy/spark:3.5         spark-master        Up
spark-worker        bitnamilegacy/spark:3.5         spark-worker        Up
warehouse           postgres:16                     warehouse           Up (healthy)
```

`airflow-init` showing **Exited (0)** is correct — it creates Airflow's database tables and the `admin` user, then stops. The webserver can take 1–2 minutes to become `(healthy)`.

✅ **Checkpoint:** seven services `Up`, `airflow-init` `Exited (0)`.

### Step 8 — Open the three web UIs

| Service | URL | Login | You should see |
|---|---|---|---|
| MinIO console | http://localhost:9001 | `minioadmin` / `minioadmin123` | Object Browser, no buckets yet |
| Airflow | http://localhost:8080 | `admin` / `admin` | "DAGs" page, empty list |
| Spark master | http://localhost:8081 | none | "Alive Workers: 1" |

**Expected output:** all three pages load and logins succeed.

✅ **Checkpoint:** you logged in to MinIO and Airflow, and the Spark page shows one alive worker.

### Step 9 — Verify the tools inside the Airflow container

Every job in SK-03 runs **inside** the `airflow-scheduler` container (that's where Python, Spark, dbt and the env vars live). `docker exec -it <container> <command>` runs a command there.

```powershell
docker exec -it airflow-scheduler python -c "import pyspark, boto3; print('pyspark', pyspark.__version__)"
docker exec -it airflow-scheduler spark-submit --version
docker exec -it airflow-scheduler dbt --version
docker exec -it airflow-scheduler bash -c 'echo $MINIO_ENDPOINT $WAREHOUSE_HOST'
```

(Single quotes matter: in double quotes PowerShell would try to expand `$MINIO_ENDPOINT` itself, find nothing, and pass an empty string.)

**Expected output** (version banners abbreviated):

```
pyspark 3.5.1
...   version 3.5.1
...
Core:
  - installed: 1.8.x
Plugins:
  - postgres: 1.8.x
http://minio:9000 warehouse
```

✅ **Checkpoint:** PySpark `3.5.1`, dbt Core `1.8.x` with the `postgres` plugin, and both env vars print.

### Step 10 — Verify the warehouse bootstrap

```powershell
docker exec -it warehouse psql -U dwh_user -d warehouse -c "\dn"
docker exec -it warehouse psql -U dwh_user -d warehouse -c "\dt staging.*"
```

**Expected output:**

```
      List of schemas
   Name    |       Owner
-----------+-------------------
 analytics | dwh_user
 public    | pg_database_owner
 staging   | dwh_user
(3 rows)

             List of relations
 Schema  |     Name     | Type  |  Owner
---------+--------------+-------+----------
 staging | daily_orders | table | dwh_user
(1 row)
```

✅ **Checkpoint:** schemas `staging` and `analytics` exist, and `staging.daily_orders` is listed.

### Step 11 — Prove containers can talk to each other

```powershell
docker exec -it airflow-scheduler curl -s -o /dev/null -w "minio HTTP %{http_code}\n" http://minio:9000/minio/health/live
docker exec -it airflow-scheduler python -c "import psycopg2; psycopg2.connect(host='warehouse', dbname='warehouse', user='dwh_user', password='dwh_pass').close(); print('WAREHOUSE_OK')"
```

**Expected output:**

```
minio HTTP 200
WAREHOUSE_OK
```

Both commands ran **inside** a container and used **service names**. Try the first one with `localhost:9000` instead of `minio:9000` — it fails, because inside the container `localhost` is the container itself.

✅ **Checkpoint:** `minio HTTP 200` and `WAREHOUSE_OK`.

### Step 12 — Learn the daily commands

```powershell
docker compose up -d                       # start (fast after the first time)
docker compose stop                        # pause everything, keep all data — end of day
docker compose down                        # remove containers, KEEP data volumes
docker compose down -v                     # full reset: DELETES buckets, warehouse, Airflow history
docker compose logs -f airflow-scheduler   # follow one service's log (Ctrl+C to stop following)
```

Run `docker compose stop`, then `docker compose up -d`, then `docker compose ps`.

**Expected output:** the same table as Step 7.

✅ **Checkpoint:** you can stop and restart the platform without losing anything.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `docker: command not found` / "cannot connect to the Docker daemon" | Docker Desktop not running, or old PowerShell window | Start Docker Desktop, wait for "Engine running", open a NEW PowerShell |
| `port is already allocated` (8080, 5433, 9000...) | Another program uses the port (e.g. a Postgres on 5433) | `netstat -ano \| findstr :5433` to find it; stop it, or change only the left side of the mapping (`"5434:5432"`) and use the new port from Windows |
| Build fails at `apt-get` or `curl ... maven` | Network / proxy / VPN | Retry off VPN; set the proxy in Docker Desktop → Settings → Resources → Proxies |
| `pull access denied for minio/minio` | You copied an old compose file | Use `bitnamilegacy/minio:2025.4.22` exactly as in Step 6 |
| `airflow-webserver` keeps restarting | `airflow-init` failed | `docker compose logs airflow-init`; then `docker compose down` and `docker compose up -d` |
| `staging.daily_orders` missing in Step 10 | The volume already existed, so `warehouse_init.sql` never ran | `docker exec -i warehouse psql -U dwh_user -d warehouse < sql\warehouse_init.sql` |
| Laptop very slow | Too little memory for Docker | Revisit `.wslconfig`; `docker compose stop spark-master spark-worker` when you aren't in Lab 02 |
| Edits to the Dockerfile have no effect | Cached image | `docker compose up -d --build` |
| Strange file-lock / permission errors | Project inside OneDrive | Move the project to `C:\DataEngineering\sk03` |

## Key takeaways

- The whole platform is three text files: `docker-compose.yml`, `docker\Dockerfile`, `sql\warehouse_init.sql`. Anyone can rebuild it identically — infrastructure as code.
- Inside containers use service names (`minio:9000`, `warehouse:5432`); from Windows use `localhost` and the published port (`9001`, `8080`, `8081`, `5433`).
- Every SK-03 job runs inside `airflow-scheduler` via `docker exec` — that is where Spark, dbt, and the credentials live.
- Credentials sit in the compose file only because this is a local lab; production uses a secrets manager.
- `docker compose ps` and `docker compose logs <service>` are always your first debugging moves.

## Next

[Lab 01 — MinIO Data Lake and Sample Data](Lab_01_MinIO_Data_Lake_and_Sample_Data.md): create UrbanGear's `urbangear-data-lake` bucket and generate the raw JSON order files the guided lab processes.
