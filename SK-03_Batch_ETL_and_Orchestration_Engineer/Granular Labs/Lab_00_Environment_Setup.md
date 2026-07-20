# Lab 00 — Environment Setup: Docker, Spark, Airflow, dbt, and MinIO on Windows 11

> **Module:** SK-03 Batch ETL and Orchestration Engineer
> **Time estimate:** 3–5 hours (most of it is downloads and verification — don't rush it)
> **Prerequisites:** SK-01 (Python) and SK-02 (SQL) completed. No Docker, Spark, Airflow, or dbt experience required — we explain everything.

---

## What this lab is for

Every other lab in this module runs on the environment you build **here, once**. By the end of this lab you will have a complete, production-shaped batch data platform running on your own laptop:

- **Docker Desktop** — runs everything below in isolated containers
- **MinIO** — an S3-compatible object store (your local stand-in for Amazon S3)
- **Apache Spark** — a distributed data processing engine (master + worker)
- **Apache Airflow** — a workflow orchestrator (webserver + scheduler)
- **PostgreSQL ×2** — one as Airflow's internal database, one as your analytics **warehouse**
- **dbt** — a SQL transformation tool (installed inside the Airflow container)

Nothing touches the cloud. Nothing costs money. Everything is deletable with one command.

**Do not skip verification steps.** Half of all "my pipeline is broken" problems in later labs are actually environment problems that a five-second check here would have caught.

---

## 1. Environment Setup

### 1.1 What you need before starting

| Requirement | Minimum | How to check |
|---|---|---|
| Windows 11 (or Windows 10 21H2+) | 64-bit, virtualization enabled | `winver` in Start menu |
| RAM | 16 GB strongly recommended (8 GB works but is tight) | Task Manager → Performance |
| Free disk space | 25 GB | File Explorer → This PC |
| CPU virtualization | Enabled in BIOS | Task Manager → Performance → CPU → "Virtualization: Enabled" |
| Admin rights | Required for installs | You'll know when an installer asks |

**If "Virtualization: Disabled"** — reboot into BIOS/UEFI (usually F2, F10, or Del at boot) and enable "Intel VT-x" / "AMD-V" / "SVM Mode". Docker cannot run without it.

> **macOS/Linux readers:** install Docker Desktop for Mac or Docker Engine for Linux; skip the WSL2 sections. Everything from section 1.5 onward is identical because it all runs inside containers.

### 1.2 What is WSL2, and why does Docker need it?

**WSL2 (Windows Subsystem for Linux, version 2)** is a lightweight Linux virtual machine built into Windows. Docker containers are a Linux technology — they need a Linux kernel to run. On Windows, Docker Desktop uses WSL2 as that kernel. You install WSL2 once, and Docker manages it for you afterwards; you rarely interact with it directly.

**Install WSL2.** Open **PowerShell as Administrator** (Start → type "PowerShell" → right-click → Run as administrator):

```powershell
wsl --install
```

- **What this does:** enables the WSL feature, installs the Linux kernel, and installs Ubuntu as a default distribution.
- **Expected output:** progress messages ending with a prompt to **restart your computer**. Restart.
- **After restart:** an Ubuntu window may open asking you to create a Linux username/password. Pick anything memorable (this is separate from your Windows login). You can close the window afterwards.

**Verify:**

```powershell
wsl --status
```

Expected output includes `Default Version: 2`. If it says version 1, run:

```powershell
wsl --set-default-version 2
```

**Common problems:**

| Symptom | Cause | Fix |
|---|---|---|
| `wsl --install` says "not recognized" | Old Windows 10 build | Run Windows Update; you need 21H2 or later |
| Error 0x80370102 | Virtualization disabled | Enable VT-x/AMD-V in BIOS (see 1.1) |
| "WSL 2 requires an update to its kernel component" | Kernel out of date | `wsl --update` in admin PowerShell |

### 1.3 Install Docker Desktop

**What is Docker?** Docker packages software into **containers** — self-contained boxes holding an application plus everything it needs (OS libraries, Python, Java, config). A container runs identically on your laptop and on a production server. That is why the entire modern data stack ships as Docker images: instead of "install Java 17, then Spark 3.5, then set six environment variables," you run one command and get a working Spark.

**An image is the recipe; a container is the running cake.** You download images once; you start/stop containers as often as you like.

1. Download Docker Desktop from **https://www.docker.com/products/docker-desktop/** (choose Windows — AMD64 for nearly all machines).
2. Run the installer. When asked, **keep "Use WSL 2 instead of Hyper-V" checked**.
3. Restart if prompted, then launch **Docker Desktop** from the Start menu.
4. Accept the service agreement. You may skip sign-in — no account is needed for this module.
5. Wait for the whale icon in the system tray to stop animating and the Docker Desktop window to say **"Engine running"**.

**Configure resources.** Docker Desktop → Settings (gear icon) → **Resources**. With the WSL2 backend, memory is managed by Windows automatically, but you should cap it so Spark doesn't starve your laptop. Create a file named `.wslconfig` in your Windows home folder (`C:\Users\<YourName>\.wslconfig`) with:

```ini
[wsl2]
memory=10GB
processors=4
```

(Use `memory=6GB` on an 8 GB machine.) Then in PowerShell: `wsl --shutdown` and restart Docker Desktop.

**Verify Docker works** (regular PowerShell, no admin needed):

```powershell
docker --version
docker run --rm hello-world
```

- **Expected output:** a version like `Docker version 27.x` (any recent version is fine — "latest stable" is the rule in this module), then a message beginning **"Hello from Docker!"**.
- **What just happened:** Docker downloaded a tiny image, ran it as a container, printed a message, and (because of `--rm`) deleted the container. You just ran your first container.

**Common problems:**

| Symptom | Fix |
|---|---|
| `docker: command not found` | Docker Desktop isn't running, or you didn't restart PowerShell after install. Launch Docker Desktop, open a NEW PowerShell window. |
| "Docker Desktop is unable to start" | Settings → General → confirm "Use the WSL 2 based engine" is checked; run `wsl --update` |
| Painfully slow downloads | Corporate proxy/VPN — try off VPN, or configure the proxy in Docker Desktop settings |

### 1.4 Create the project folder structure

All labs use **one** project folder. We recommend keeping it out of OneDrive-synced folders (file-sync tools fight with Docker volume mounts).

```powershell
mkdir C:\projects\sk03-batch-etl
cd C:\projects\sk03-batch-etl
mkdir dags, jobs, dbt, data, logs, plugins, config, docker
```

**What each folder is for (you'll fill them across the labs):**

| Folder | Purpose | Used from |
|---|---|---|
| `dags\` | Airflow DAG definitions (Python files that describe workflows) | Lab 04 |
| `jobs\` | PySpark job scripts | Lab 02 |
| `dbt\` | Your dbt project (SQL models and tests) | Lab 06 |
| `data\` | Sample input files you generate | Lab 01 |
| `logs\` | Airflow task logs (mounted so you can read them from Windows) | Lab 04 |
| `plugins\` | Airflow plugins (stays mostly empty) | — |
| `config\` | Shared configuration files | Lab 05 |
| `docker\` | The custom Airflow Dockerfile (next section) | now |

**Verify:** `dir` shows all eight folders.

### 1.5 The docker-compose file — your entire platform in one file

**What is docker-compose?** Running six containers by hand, wiring their networks and ports, would be miserable. **Docker Compose** lets you declare the whole stack in one YAML file and start it with one command. The file below **is your data platform**. Read the comments — every line teaches you something about how these tools connect in production.

Create `C:\projects\sk03-batch-etl\docker-compose.yml` in a text editor (VS Code recommended — `code .` from the project folder if you have it). Type or paste **exactly** this:

```yaml
# SK-03 local batch data platform
# Start:  docker compose up -d
# Stop:   docker compose down          (keeps data)
# Reset:  docker compose down -v       (DELETES all data volumes)

x-airflow-common: &airflow-common
  build: ./docker
  environment: &airflow-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@airflow-db/airflow
    AIRFLOW__CORE__LOAD_EXAMPLES: "false"
    AIRFLOW__WEBSERVER__EXPOSE_CONFIG: "true"
    AIRFLOW__CORE__DEFAULT_TIMEZONE: utc
    # MinIO credentials — in real production these come from a secrets manager,
    # never from a compose file. See "Production Engineering Practices" below.
    MINIO_ENDPOINT: http://minio:9000
    MINIO_ACCESS_KEY: minioadmin
    MINIO_SECRET_KEY: minioadmin123
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
    - ./config:/opt/airflow/shared_config
  depends_on:
    airflow-db:
      condition: service_healthy

services:

  # ------------------------------------------------------------------
  # MinIO — S3-compatible object storage. Simulates Amazon S3 locally.
  # API on 9000 (what code talks to), web console on 9001 (what YOU use).
  # ------------------------------------------------------------------
  minio:
    image: minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - minio-data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ------------------------------------------------------------------
  # Airflow's internal metadata database. Airflow stores DAG runs,
  # task states, connections, and variables here. You never query it.
  # ------------------------------------------------------------------
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

  # ------------------------------------------------------------------
  # The analytics WAREHOUSE — where dbt builds models and analysts query.
  # In production this would be Redshift/Snowflake/BigQuery.
  # Exposed on host port 5433 so you can connect with any SQL client.
  # ------------------------------------------------------------------
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dwh_user -d warehouse"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ------------------------------------------------------------------
  # Spark master — coordinates work. Web UI on host port 8081.
  # ------------------------------------------------------------------
  spark-master:
    image: bitnami/spark:3.5
    container_name: spark-master
    environment:
      SPARK_MODE: master
    ports:
      - "8081:8080"   # Spark master web UI (8080 inside; 8081 on your machine
                      # because Airflow's UI takes 8080)
      - "7077:7077"   # where workers and jobs connect
    volumes:
      - ./jobs:/opt/jobs
      - ./data:/opt/data

  # ------------------------------------------------------------------
  # Spark worker — does the actual data processing.
  # ------------------------------------------------------------------
  spark-worker:
    image: bitnami/spark:3.5
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
      - ./data:/opt/data

  # ------------------------------------------------------------------
  # Airflow: one-time init (creates DB tables + admin user), then
  # webserver (UI on 8080) and scheduler (runs your DAGs).
  # ------------------------------------------------------------------
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
          --role Admin --email admin@example.com
    restart: "no"

  airflow-webserver:
    <<: *airflow-common
    container_name: airflow-webserver
    command: webserver
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
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

**Things worth understanding in that file (read this — it pays off all module):**

- **`x-airflow-common` / `&airflow-common` / `<<:`** — YAML anchors. We define the Airflow configuration once and reuse it for init, webserver, and scheduler. Same principle as a function in code: don't repeat yourself.
- **Service names are hostnames.** Inside the Docker network, containers reach each other by service name: Airflow talks to `http://minio:9000`, dbt talks to host `warehouse`. From **Windows**, you use `localhost` with the published port instead (e.g., `localhost:9001` for the MinIO console). Mixing these up is the #1 beginner error in this module.
- **Volumes** (`./dags:/opt/airflow/dags`) mount your Windows folders into containers. Edit a DAG in VS Code on Windows → Airflow inside the container sees it within seconds. **Named volumes** (`minio-data:`) persist databases and buckets across restarts.
- **Healthchecks + `depends_on: condition:`** — Airflow refuses to start until its database answers. This is real production thinking: never assume a dependency is up; check.

### 1.6 The custom Airflow image

The stock Airflow image doesn't include Java (needed by PySpark), PySpark itself, dbt, or the S3 client library. We extend it. Create `C:\projects\sk03-batch-etl\docker\Dockerfile`:

```dockerfile
FROM apache/airflow:2.9.3-python3.11

# Java is required by PySpark (Spark runs on the JVM)
USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-17-jre-headless \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

# Python packages must be installed as the airflow user, never as root
USER airflow
RUN pip install --no-cache-dir \
    pyspark==3.5.1 \
    dbt-postgres==1.8.2 \
    boto3==1.34.131 \
    apache-airflow-providers-amazon==8.24.0
```

**Why one image for Airflow + PySpark + dbt?** In large production systems, Spark jobs run on a dedicated cluster (EMR, Databricks, Kubernetes) and Airflow only *submits* them. Locally, we take a pragmatic, widely-used shortcut: Airflow tasks run `spark-submit` in local mode inside the Airflow container. You still get the standalone Spark cluster (master + worker) for Labs 02–03 so you learn the real cluster model and the Spark UI. Lab 05 explains this trade-off in depth.

**Pin your versions.** Notice every package has an exact version (`2.9.3`, `3.5.1`, `1.8.2`). Unpinned versions are how "it worked yesterday" happens. If a newer stable version exists when you read this, these pinned versions still work together — use them for the module.

### 1.7 First launch

From `C:\projects\sk03-batch-etl` in PowerShell:

```powershell
docker compose up -d --build
```

- **What this does:** builds your custom Airflow image (first time: 5–10 minutes), downloads MinIO/Postgres/Spark images (first time: several GB), then starts everything in the background (`-d` = detached).
- **Expected output:** build logs, then lines like `✔ Container minio  Started` for each service. `airflow-init` will show `Exited (0)` — **that is correct**; it runs once and stops.

**Verify everything is healthy:**

```powershell
docker compose ps
```

Expected: `minio`, `airflow-db`, `warehouse`, `spark-master`, `spark-worker`, `airflow-webserver`, `airflow-scheduler` all `Up` (db/minio/warehouse/webserver showing `(healthy)`); `airflow-init` shows `Exited (0)`.

Now open each UI in your browser and confirm you can log in:

| Service | URL | Login | You should see |
|---|---|---|---|
| MinIO console | http://localhost:9001 | `minioadmin` / `minioadmin123` | Object Browser, "no buckets" |
| Airflow | http://localhost:8080 | `admin` / `admin` | Empty DAG list |
| Spark master UI | http://localhost:8081 | none | "Workers (1)" listed, status ALIVE |

**Verify the warehouse** (uses the Postgres client inside the container, so you don't need to install anything on Windows):

```powershell
docker exec -it warehouse psql -U dwh_user -d warehouse -c "SELECT version();"
```

Expected: a row starting `PostgreSQL 16...`.

**Verify PySpark + Java inside Airflow:**

```powershell
docker exec -it airflow-scheduler python -c "import pyspark; print(pyspark.__version__)"
docker exec -it airflow-scheduler dbt --version
```

Expected: `3.5.1`, then dbt Core `1.8.x` with the `postgres` plugin listed.

### 1.8 Troubleshooting first launch

| Symptom | Cause | Fix |
|---|---|---|
| `port is already allocated` on 8080 | Something else uses the port | Find it: `netstat -ano \| findstr :8080`, stop it, or change the left side of the mapping (e.g. `8085:8080`) and use that URL |
| Webserver restarts forever | init failed | `docker compose logs airflow-init` — usually the DB wasn't healthy yet; `docker compose down` then `up -d` again |
| Build fails at `apt-get` | Network/proxy | Retry off VPN; check Docker Desktop proxy settings |
| Everything is slow / laptop fan screams | Not enough RAM allocated | Revisit `.wslconfig` (1.3); close browsers; on 8 GB machines run `docker compose stop spark-worker` when not doing Spark labs |
| `docker compose` says "no configuration file" | Wrong folder | `cd C:\projects\sk03-batch-etl` first — compose reads the file from the current directory |
| Changes to Dockerfile not taking effect | Cached image | `docker compose up -d --build` (the `--build` matters) |

### 1.9 Daily driving: start, stop, reset

```powershell
docker compose up -d      # start (fast after first time)
docker compose stop       # pause everything, keep all data — do this end of day
docker compose down       # remove containers, KEEP data volumes
docker compose down -v    # full reset: DELETES buckets, warehouse, Airflow history
docker compose logs -f airflow-scheduler   # follow one service's logs (Ctrl+C to stop)
```

You will type these hundreds of times this module. `stop` at the end of each session; `up -d` at the start of the next.

---

## 2. Business Context

Why does a batch ETL engineer need this exact stack?

Picture UrbanGear, the online retailer whose pipeline you will rebuild starting in Lab 01. Every night, order files land in object storage. Something must **process** them at scale (Spark), something must **decide when and in what order** processing happens, retry failures, and alert humans (Airflow), and something must **shape the clean data into tables analysts trust** (dbt). The files live in **S3** in production — and MinIO speaks S3's exact API, so every line of code you write against MinIO works unchanged against real S3 by swapping an endpoint URL.

Who consumes the output? Analysts, finance teams, dashboards, ML models. What happens if the platform fails? Executives make decisions on stale numbers, finance closes the books wrong, and — as the archive of real incidents in this module shows — companies discover multi-million-dollar reporting gaps days late.

And why Docker? Because in industry, **nobody installs Spark on their laptop by hand anymore**. Data platforms are containerized. Being fluent in `docker compose ps` and reading container logs is as fundamental to a data engineer in 2026 as knowing SQL.

---

## 3. Concept Explanation

**Containers vs. virtual machines.** A VM emulates an entire computer (its own OS kernel — heavy, slow to boot). A container shares the host's Linux kernel and isolates only the filesystem and processes — starts in seconds, uses megabytes. That is why we can run seven services on one laptop.

**Object storage vs. filesystems.** MinIO/S3 store *objects* (a blob + a key like `raw/orders/date=2026-07-01/orders.csv`) in *buckets*. There are no real folders — the `/` in keys is a naming convention that tools render as folders. Objects are immutable: you overwrite whole objects, never edit bytes in place. This shapes how batch pipelines write data (Lab 03).

**Why simulate S3 instead of using S3?** (a) Zero cost and zero risk of a surprise bill; (b) works offline; (c) identical API means all learning transfers; (d) you can destroy and rebuild in seconds. The one thing MinIO can't teach you is IAM permissions — that's SK-04's job.

**Alternatives you'd hear about at work:** LocalStack (simulates many AWS services, heavier), Azurite (Azure Blob), plain shared folders (no S3 API, teaches nothing transferable). MinIO is itself used *in production* by many companies as on-prem object storage — it is not a toy.

**The stack's division of labor** — memorize this sentence: *Spark computes, Airflow coordinates, dbt transforms-and-tests inside the warehouse, MinIO/S3 stores files, Postgres serves queries.*

```mermaid
flowchart LR
    A[Source files] -->|land in| B[(MinIO<br/>raw bucket)]
    B --> C[Spark job<br/>clean + validate]
    C -->|Parquet| D[(MinIO<br/>processed bucket)]
    D --> E[Load to warehouse]
    E --> F[(Postgres<br/>warehouse)]
    F --> G[dbt models + tests]
    G --> H[Analysts / BI]
    I[Airflow] -.orchestrates.-> C
    I -.orchestrates.-> E
    I -.orchestrates.-> G
```

---

## 4. Step-by-Step Implementation

You already did the heavy lifting in section 1. This section is one guided smoke test that touches **every** component, so you know the whole platform works end-to-end before Lab 01.

### Step 4.1 — Create a bucket and upload a file to MinIO

1. Open http://localhost:9001, log in (`minioadmin` / `minioadmin123`).
2. Click **Create Bucket** → name it `smoke-test` → Create.
3. On Windows, create a tiny file:

```powershell
"id,name`n1,alpha`n2,beta" | Out-File -Encoding utf8 C:\projects\sk03-batch-etl\data\smoke.csv
```

4. In the MinIO console, open the `smoke-test` bucket → **Upload** → choose `smoke.csv`.

**Verify:** the object browser shows `smoke.csv`, size ~30 B.
**Common mistake:** `Out-File` without `-Encoding utf8` writes UTF-16 on Windows PowerShell 5.1, which Spark reads as garbage. Always pass `-Encoding utf8`.

### Step 4.2 — Read that file from Spark

Run a Spark shell **inside** the container, configured to talk to MinIO via the `s3a://` connector (the Hadoop library Spark uses for S3-compatible storage):

```powershell
docker exec -it spark-master spark-shell `
  --packages org.apache.hadoop:hadoop-aws:3.3.4,com.amazonaws:aws-java-sdk-bundle:1.12.262 `
  --conf spark.hadoop.fs.s3a.endpoint=http://minio:9000 `
  --conf spark.hadoop.fs.s3a.access.key=minioadmin `
  --conf spark.hadoop.fs.s3a.secret.key=minioadmin123 `
  --conf spark.hadoop.fs.s3a.path.style.access=true
```

- **Why these confs:** `endpoint` points s3a at MinIO instead of AWS; `path.style.access=true` is required because MinIO doesn't use AWS's bucket-as-subdomain URLs. You will see these five settings in every lab — Lab 02 wraps them in a helper so you stop typing them.
- First run downloads the two JAR packages (~1 min).

At the `scala>` prompt:

```scala
spark.read.option("header", "true").csv("s3a://smoke-test/smoke.csv").show()
```

**Expected output:**

```
+---+-----+
| id| name|
+---+-----+
|  1|alpha|
|  2| beta|
+---+-----+
```

Type `:quit` to exit.

**Troubleshooting:** `UnknownHostException: minio` → you ran the shell on Windows instead of inside the container (only containers resolve the name `minio`). `403 Forbidden` → credential typo. `NoSuchBucket` → bucket name typo.

### Step 4.3 — Prove Airflow can run a task

1. Create `C:\projects\sk03-batch-etl\dags\smoke_test_dag.py`:

```python
"""Lab 00 smoke test: proves the scheduler executes tasks and can reach MinIO."""
from datetime import datetime
from airflow import DAG
from airflow.operators.bash import BashOperator

with DAG(
    dag_id="lab00_smoke_test",
    start_date=datetime(2026, 1, 1),
    schedule=None,          # manual trigger only
    catchup=False,
    tags=["lab00"],
) as dag:
    check_minio = BashOperator(
        task_id="check_minio_reachable",
        bash_command="curl -sf http://minio:9000/minio/health/live && echo MINIO_OK",
    )
```

2. Wait up to 60 seconds, refresh http://localhost:8080 — `lab00_smoke_test` appears (the scheduler scans the `dags/` folder every ~30s).
3. Click the DAG → press the **▶ Trigger** button (top right).
4. Click the green square that appears → **Logs**.

**Expected:** log ends with `MINIO_OK` and `Marking task as SUCCESS`. You just ran your first orchestrated task, and it proved container-to-container networking works.

**Common mistakes:** DAG not appearing → Python syntax error; check `docker compose logs airflow-scheduler` for an import error, or the file was saved outside `dags\`. Task fails with `curl: command not found` → you edited the Dockerfile and removed curl inheritance; rebuild.

### Step 4.4 — Prove dbt can reach the warehouse

```powershell
docker exec -it airflow-scheduler bash -c "dbt debug --profiles-dir /opt/airflow/dbt || true"
```

Right now this **fails** with "Could not find profile" — expected, you haven't created a dbt project yet (Lab 06 does). What matters is that the `dbt` command runs. Now prove connectivity the direct way:

```powershell
docker exec -it airflow-scheduler python -c "import psycopg2; psycopg2.connect(host='warehouse', dbname='warehouse', user='dwh_user', password='dwh_pass').close(); print('WAREHOUSE_OK')"
```

**Expected:** `WAREHOUSE_OK`.

### Step 4.5 — Clean up the smoke test

Delete the `smoke-test` bucket in the MinIO console (bucket → Manage → Delete) and delete `dags\smoke_test_dag.py`. Keeping test artifacts around is how confusion breeds — a habit worth building on day one.

---

## 5. Production Engineering Practices

Even a setup lab carries production lessons. Three for today:

**1. Infrastructure as code.** Your entire platform is two text files (`docker-compose.yml`, `Dockerfile`). You could email them to a teammate and they'd have an identical environment in ten minutes. That property — *environments are reproducible from version-controlled files* — is the foundation of everything in SK-04 (Terraform is this idea applied to AWS). **Failure story:** a team whose "staging" server was configured by hand over two years could never reproduce a production bug locally; every hotfix was tested in production. Don't be that team. Consider `git init` in your project folder right now.

**2. Secrets don't belong in config files.** We put `minioadmin123` straight into the compose file because this is a throwaway local lab — and we flagged it with a comment. In production, credentials come from a secrets manager (AWS Secrets Manager, Vault) or at minimum environment files excluded from git. **Failure story:** uncountable real breaches started with AWS keys committed to a public GitHub repo; bots scan for them and start crypto-miners within minutes. Lab 05 introduces Airflow Connections as the proper home for credentials in DAGs.

**3. Healthchecks and explicit dependencies.** Our compose file makes Airflow wait for a *healthy* database, not just a *started* container. Production systems fail most often at startup and during dependency outages; declaring "what must be true before I run" is the same thinking behind Airflow sensors (Lab 05). **Failure story:** a pipeline that "worked on the engineer's machine" failed every Monday in production because a database restarted Sunday nights and the app connected before it accepted connections — a one-line healthcheck fix, found after three weeks of Monday pages.

---

## 6. Reflection

**What you built:** a seven-container local data platform — object storage (MinIO), compute (Spark), orchestration (Airflow), warehouse (Postgres), transformation tooling (dbt) — from two version-controlled files, and smoke-tested every path through it.

**Why it matters:** every subsequent lab assumes this environment. More importantly, you now hold the mental model of the modern batch stack: *storage, compute, orchestration, and transformation are separate tools with clean seams*, which is exactly how real platforms — cloud or on-prem — are assembled.

### Interview questions (with model answers)

1. **What's the difference between a Docker image and a container?** An image is an immutable packaged filesystem + startup command; a container is a running (or stopped) instance of an image. One image, many containers.
2. **Why do containers start so much faster than VMs?** Containers share the host Linux kernel and isolate only processes/filesystem/network; VMs boot an entire guest OS.
3. **What is object storage and how does it differ from a filesystem?** Flat key→blob storage with immutable objects accessed over HTTP APIs; no true directories, no in-place edits, no POSIX semantics. Optimized for huge scale and throughput, not low-latency small updates.
4. **How can code written against MinIO run unchanged against Amazon S3?** MinIO implements the S3 API. Only the endpoint URL and credentials change; SDKs (boto3, Hadoop s3a) accept a custom endpoint parameter.
5. **What does `s3a://` mean in a Spark path?** It selects the Hadoop S3A connector — the maintained Hadoop filesystem implementation for S3-compatible stores (its predecessors `s3://`/`s3n://` in Hadoop are obsolete).
6. **In docker-compose, when do you use the service name vs. localhost?** Service names resolve only on the Docker network (container→container). From the host you use `localhost:<published port>`. A classic bug is a container trying to call `localhost:9000` and reaching itself.
7. **Why does Airflow need its own database?** It persists all state there — DAG runs, task instances, retries, connections, variables. The scheduler and webserver are stateless; the metadata DB is the source of truth.
8. **Why pin dependency versions in a Dockerfile?** Reproducibility. Unpinned builds can silently pull incompatible versions weeks later, producing "nothing changed but it broke" incidents.

**Interview traps to watch for:**
- "Is MinIO a database?" No — it's object storage; there is no query engine, no transactions over rows.
- "Docker makes things faster, right?" Not inherently — it makes them *reproducible and isolated*. Performance is roughly native on Linux, slightly taxed via WSL2.
- Interviewers love asking why `localhost` fails inside a container — be ready with the network-namespace answer from Q6.

**Key takeaways:**
- The whole platform is code: `docker-compose.yml` + `Dockerfile`.
- Service names inside the network, `localhost:port` from Windows.
- MinIO = S3 API locally; everything you write transfers to AWS.
- `docker compose ps` and `docker compose logs <service>` are your first debugging moves — always.
- Never skip verification. Every lab, every step.

**Next:** [Lab 01 — Docker and MinIO Fundamentals](Lab_01_Docker_and_MinIO_Fundamentals.md), where you meet UrbanGear's data, script MinIO with Python (boto3), and learn the bucket layout the whole module builds on.
