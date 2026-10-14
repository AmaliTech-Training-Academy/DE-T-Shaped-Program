# Lab 00 — Environment Setup

> **Goal:** Run PostgreSQL 16 in Docker on Windows 11, connect to it with `psql`, and create the project folder that every later lab and the guided lab use.
> **Time:** 60–90 min | **Difficulty:** Beginner | **Prerequisites:** Windows 11 PC with admin rights, ~10 GB free disk space
> **Guided lab coverage:** PREREQUISITES section of `SK-02_End_to_End_Guided_Lab.md` (PostgreSQL 14+ and a SQL client)

## Learning objectives

By the end of this lab you can:

- Explain the words **database server**, **client**, **port**, **container**, **image** and **volume**.
- Create the SK-02 project folder `C:\DataEngineering\sk02` (and explain why it must not live in OneDrive).
- Start PostgreSQL 16 with Docker Compose and check that it is healthy.
- Open an interactive `psql` session and run a `.sql` script file — the two commands every later lab uses.
- Stop, start and (when you really mean it) reset the database.

**Key terms**

| Term | Meaning |
|---|---|
| **PostgreSQL** ("Postgres") | A free, open-source relational database server. Its SQL is close to the ANSI standard, so what you learn transfers to Snowflake, Redshift and BigQuery. |
| **Database server** | A program that runs in the background, stores data and answers SQL requests. |
| **Client** | A program that sends SQL to the server. We use **psql** (command line); a GUI client is optional. |
| **Port** | A numbered "door" on a machine. PostgreSQL listens on **5432**. |
| **Docker image** | A packaged, read-only template of software — here `postgres:16`. |
| **Container** | A running instance of an image — an isolated mini-machine. Ours is called `sk02-postgres`. |
| **Volume** | Docker-managed storage that survives container restarts. Our data lives in volume `sk02_pgdata`. |

**Why Docker?** One command gives every trainee the *identical* PostgreSQL 16 — same version, same settings, same credentials — and SK-03 uses the same approach. Nothing is installed into Windows itself, and a broken database can be thrown away and recreated in a minute.

## Steps

### Step 1 — Create the project folder

Open **PowerShell** (press `Win`, type `powershell`, Enter) and run:

```powershell
New-Item -ItemType Directory -Force C:\DataEngineering\sk02\sql
Set-Location C:\DataEngineering\sk02
Get-ChildItem
```

**Why `C:\DataEngineering` and not Documents/Desktop?** On most company laptops Documents and Desktop are synced by **OneDrive**. OneDrive locks files while syncing and Docker often cannot share OneDrive folders with containers — you get empty folders and "file not found" errors that are very hard to diagnose. Never put this project under a path containing `OneDrive`.

**Expected output** (the last command):

```
    Directory: C:\DataEngineering\sk02

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----        ...                                  sql
```

✅ **Checkpoint:** `C:\DataEngineering\sk02\sql` exists and your PowerShell prompt shows `PS C:\DataEngineering\sk02>`.

### Step 2 — Install Docker Desktop

1. Docker Desktop needs **WSL 2** (Windows Subsystem for Linux). In an **admin** PowerShell run `wsl --install`, then reboot if asked.
2. Install Docker Desktop: either download it from <https://www.docker.com/products/docker-desktop/> or run (admin PowerShell):

   ```powershell
   winget install -e --id Docker.DockerDesktop
   ```

3. Start **Docker Desktop** from the Start menu and wait until it says *Engine running*.
4. In a normal PowerShell window verify:

```powershell
docker --version
docker compose version
docker run --rm hello-world
```

**Expected output** (versions may be newer):

```
Docker version 27.x.x, build ...
Docker Compose version v2.x.x
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

✅ **Checkpoint:** `hello-world` prints *Hello from Docker!*

### Step 3 — Write the Docker Compose file

Create `C:\DataEngineering\sk02\docker-compose.yml` (in VS Code or Notepad) with exactly this content:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: sk02-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres     # local learning container only — never reuse in real systems
      POSTGRES_DB: freshmart
    ports:
      - "5432:5432"
    volumes:
      - sk02_pgdata:/var/lib/postgresql/data   # the database files (survive restarts)
      - ./sql:/sql                             # your scripts folder, visible inside the container as /sql

volumes:
  sk02_pgdata:
```

**What each part does:**

- `image: postgres:16` — PostgreSQL major version 16 (the guided lab needs 14+).
- `POSTGRES_DB: freshmart` — creates the `freshmart` database on first start. Every lab and the guided lab use it.
- `ports: "5432:5432"` — makes the container's port 5432 reachable on your PC as `localhost:5432` (for GUI clients).
- `./sql:/sql` — **the important one:** a file you save as `C:\DataEngineering\sk02\sql\abc.sql` appears inside the container as `/sql/abc.sql`.

Check that the file is where Docker Compose will look for it:

```powershell
Get-ChildItem C:\DataEngineering\sk02 | Select-Object Name
```

**Expected output:**

```
Name
----
sql
docker-compose.yml
```

✅ **Checkpoint:** `Get-Content C:\DataEngineering\sk02\docker-compose.yml` prints the YAML exactly as above (indentation matters in YAML).

### Step 4 — Start PostgreSQL

```powershell
Set-Location C:\DataEngineering\sk02
docker compose up -d
docker ps --filter name=sk02-postgres
```

The first run downloads the image (~400 MB) — give it a few minutes.

**Expected output** (last command):

```
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                    NAMES
xxxxxxxxxxxx   postgres:16   "docker-entrypoint.s…"   10 seconds ago   Up 9 seconds    0.0.0.0:5432->5432/tcp   sk02-postgres
```

Confirm the server finished starting:

```powershell
docker exec sk02-postgres pg_isready -U postgres
```

**Expected output:**

```
/var/run/postgresql:5432 - accepting connections
```

✅ **Checkpoint:** `STATUS` says `Up` and `pg_isready` says `accepting connections`.

### Step 5 — Connect interactively with psql

`psql` is already inside the container, so you don't install anything on Windows:

```powershell
docker exec -it sk02-postgres psql -U postgres -d freshmart
```

(`-it` = interactive terminal; `-U postgres` = log in as user postgres; `-d freshmart` = open database freshmart.) Your prompt changes to `freshmart=#`. Now run:

```sql
SELECT version();
```

**Expected output** (minor version may differ):

```
                                                       version
---------------------------------------------------------------------------------------------------------------------
 PostgreSQL 16.x (Debian 16.x-1.pgdg...) on x86_64-pc-linux-gnu, compiled by gcc ...
(1 row)
```

**Type the semicolon!** psql sends nothing until it sees `;`. A `freshmart-#` prompt (note the `-`) means psql is waiting for the rest of the statement.

psql **meta-commands** (start with `\`, are not SQL, need no `;`):

| Command | Does |
|---|---|
| `\l` | List databases |
| `\dn` | List schemas |
| `\dt source_oltp.*` | List tables in schema `source_oltp` (Lab 01 onward) |
| `\d tablename` | Describe a table's columns, indexes and constraints |
| `\timing on` | Show how long each query takes |
| `\x auto` | Display wide rows vertically when needed |
| `\q` | Quit psql |

Quit with `\q`.

✅ **Checkpoint:** `SELECT version();` shows `PostgreSQL 16`.

### Step 6 — Run a script file

All real work in SK-02 is saved as numbered `.sql` files in `C:\DataEngineering\sk02\sql\` and run with one command. Create `C:\DataEngineering\sk02\sql\00_smoke_test.sql`:

```sql
-- 00_smoke_test.sql — proves create / insert / query / drop all work
DROP TABLE IF EXISTS smoke_test;
CREATE TABLE smoke_test (
    id         SERIAL PRIMARY KEY,
    note       TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT now()
);
INSERT INTO smoke_test (note) VALUES ('PostgreSQL is alive');
SELECT id, note FROM smoke_test;
DROP TABLE smoke_test;
```

Run it:

```powershell
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/00_smoke_test.sql
```

**Expected output:**

```
psql:/sql/00_smoke_test.sql:2: NOTICE:  table "smoke_test" does not exist, skipping
DROP TABLE
CREATE TABLE
INSERT 0 1
 id |        note
----+---------------------
  1 | PostgreSQL is alive
(1 row)

DROP TABLE
```

`-v ON_ERROR_STOP=1` makes psql stop at the first error instead of carrying on — always use it for scripts.

**The two commands you will use in every lab** (copy them into your notes):

```powershell
# Interactive session
docker exec -it sk02-postgres psql -U postgres -d freshmart

# Run a script saved in C:\DataEngineering\sk02\sql\
docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/<file>.sql
```

✅ **Checkpoint:** the script prints the `PostgreSQL is alive` row.

### Step 7 — (Optional) Add a GUI client

The guided lab accepts psql, DBeaver, pgAdmin or the VS Code SQL extension. If you want a GUI, install **DBeaver Community** (`winget install -e --id dbeaver.dbeaver`) or **pgAdmin 4** and create a connection with:

| Setting | Value |
|---|---|
| Host | `localhost` |
| Port | `5432` |
| Database | `freshmart` |
| User / Password | `postgres` / `postgres` |

Run `SELECT current_database(), current_user;`.

**Expected output:** one row: `freshmart | postgres`.

✅ **Checkpoint:** the GUI shows the `freshmart` database (or you skipped this optional step).

### Step 8 — Stop, start and reset

```powershell
Set-Location C:\DataEngineering\sk02
docker compose stop        # stop the container; data is kept
docker compose start       # start it again
docker compose ps          # check status
```

**Expected output** (`docker compose ps`):

```
NAME            IMAGE         COMMAND                  SERVICE    CREATED         STATUS         PORTS
sk02-postgres   postgres:16   "docker-entrypoint.s…"   postgres   2 minutes ago   Up 3 seconds   0.0.0.0:5432->5432/tcp
```

**Reset everything** (deletes ALL data in the volume — only when you want a clean start):

```powershell
docker compose down -v
docker compose up -d
```

✅ **Checkpoint:** after `stop` + `start`, `pg_isready` (Step 4) again says `accepting connections`.

> **Alternative — native install:** if you cannot run Docker, install PostgreSQL **16** with the EDB installer from <https://www.postgresql.org/download/windows/> (port 5432, user `postgres`), add `C:\Program Files\PostgreSQL\16\bin` to your PATH, and run `CREATE DATABASE freshmart;` once. Then replace the two standard commands with `psql -U postgres -d freshmart` and `psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f C:\DataEngineering\sk02\sql\<file>.sql`. Everything else in SK-02 is identical.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `docker: error during connect ... The system cannot find the file specified` | Docker Desktop isn't running | Start Docker Desktop and wait for *Engine running*. |
| Docker Desktop says WSL 2 is required | WSL not installed/enabled | Admin PowerShell: `wsl --install`, reboot, restart Docker Desktop. |
| `Bind for 0.0.0.0:5432 failed: port is already allocated` | Another PostgreSQL (native install or another container) uses 5432 | Stop the other one (`Get-Service postgresql*` / `docker ps`), or change the mapping to `"5433:5432"` — psql via `docker exec` is unaffected; GUI clients then use port 5433. |
| `psql: error: ... FATAL: database "freshmart" does not exist` | The volume was created earlier without `POSTGRES_DB` | `docker compose down -v` then `docker compose up -d` (deletes data). |
| `password authentication failed` in a GUI client after you changed the password in the YAML | Environment variables only apply when the volume is first created | `docker compose down -v` and `up -d` again. |
| `/sql/00_smoke_test.sql: No such file or directory` | File saved elsewhere, wrong name, or project under OneDrive | Check `Get-ChildItem C:\DataEngineering\sk02\sql`; make sure you ran `docker compose up` from `C:\DataEngineering\sk02`. |
| psql shows `freshmart-#` and does nothing | Statement has no `;` yet | Type `;` and press Enter. |

## Key takeaways

- PostgreSQL 16 runs in container `sk02-postgres`; database `freshmart`, user/password `postgres`/`postgres`, port 5432.
- Scripts live in `C:\DataEngineering\sk02\sql\` and appear in the container as `/sql/`.
- Interactive: `docker exec -it sk02-postgres psql -U postgres -d freshmart`. Script: `docker exec sk02-postgres psql -U postgres -d freshmart -v ON_ERROR_STOP=1 -f /sql/<file>.sql`.
- Keep the project out of OneDrive. `docker compose down -v` wipes the data — use it deliberately.

## Next

[Lab 01 — Building the FreshMart Source System](Lab_01_Building_the_FreshMart_Source_System.md): create the `source_oltp` schema the guided lab analyses, and fill it with deterministic data.
