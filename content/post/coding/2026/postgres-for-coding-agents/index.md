---
title: "A Local Postgres for Coding Agents: One Role to Look, One Role to Change"
date: 2026-11-24T15:00:00+11:00
publishDate: 2026-11-24
draft: false
layout: "post"
tags: ["postgres", "docker", "lima", "claude-code", "ai", "devvm", "databases"]
categories: ["coding"]
description: "How my dev VM gives coding agents a real Postgres to explore with a read-only role and change with a separate migration role, with no passwords to paste and nothing to drift."
author: "Paul"
---

Agents write better database code when they can see the database. Read the actual schema instead of guessing it from entity classes. Try a query before it goes into code. Run a migration and look at the result. So I added a Postgres to my [dev VM](https://github.com/prule/devvm) that Claude Code can use while it works.

The catch is that "use the database" covers two very different things: looking and changing. I wanted looking to be the default and safe, and changing to be a deliberate choice. So the database has two login roles:

| Role | For | Can |
|---|---|---|
| `dev_ro` | exploring | read every table in every database, change nothing |
| `dev_rw` | migrations and seed data | own the `dev` database, create more; not a superuser |

And connecting is one of these, with no password to type:

```bash
psql service=dev_ro                      # explore
psql service=dev_rw                      # change things
psql "service=dev_ro dbname=myproject"   # another database on the same server
```

## The server

Postgres runs in Docker in the VM, published on `127.0.0.1:5432` only (Lima forwards it to the Mac at the same address). It's separate from the Supabase stack, which has its own database on port 54322. This one is just for local data.

```bash
docker inspect postgres > /dev/null 2>&1 \
  || docker run -d --name postgres --restart unless-stopped \
    -p 127.0.0.1:5432:5432 \
    -e POSTGRES_PASSWORD="$(cat /etc/devdb/postgres)" \
    -v postgres_data:/var/lib/postgresql \
    "postgres:${PARAM_POSTGRES_VERSION}"
```

Two details in there that cost a bit of reading:

- **The volume is mounted at `/var/lib/postgresql`, not `.../data`.** From Postgres 18, the official image keeps its data in a per-major subdirectory there.
- **The startup wait uses TCP, not the socket.** On first start, the image runs a socket-only server to initialise the database, then restarts. Anything sent to that first server gets cut off by the restart. Waiting on `pg_isready -h 127.0.0.1` skips past it.

## The two roles

On every boot the VM runs this against the server as the `postgres` superuser:

```sql
SELECT 'CREATE ROLE dev_rw' WHERE NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'dev_rw') \gexec
SELECT 'CREATE ROLE dev_ro' WHERE NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'dev_ro') \gexec
ALTER ROLE dev_rw LOGIN CREATEDB;   -- plus its generated password, see below
ALTER ROLE dev_ro LOGIN;            -- likewise
ALTER ROLE dev_ro SET default_transaction_read_only = on;
GRANT pg_read_all_data TO dev_ro;
SELECT 'CREATE DATABASE dev OWNER dev_rw' WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'dev') \gexec
```

The two `LOGIN` lines also set each role's password, from psql variables named `rw_pw` and `ro_pw`, which the boot fills from the generated password files.

`dev_ro` is read-only twice over. `pg_read_all_data` is a built-in role that can `SELECT` from every table in every database, and `dev_ro` has no write privileges anywhere. On top of that, its sessions start with `default_transaction_read_only = on`, so a write attempt fails with a plain "read-only transaction" error.

`dev_rw` owns `dev` and can create more databases, say one per project. It isn't a superuser, so it can't touch the server's configuration or other roles.

Everything is written to run again safely. `CREATE ROLE` and `CREATE DATABASE` have no `IF NOT EXISTS`, hence the `SELECT ... \gexec` trick, which only runs the statement when the row is missing. The `ALTER`s and `GRANT` simply re-apply. So a re-provision and a fresh empty volume both end up in the same state.

The SQL goes in through stdin, not `psql -c`, because psql only expands `:'var'` variables in input it reads. That's how the passwords get in without being pasted into the SQL.

## No passwords to paste

Passwords are generated once per VM and kept where only root can read them:

```bash
install -d -m 700 /etc/devdb
for role in postgres dev_ro dev_rw; do
  [ -s "/etc/devdb/$role" ] || (umask 077; openssl rand -hex 24 > "/etc/devdb/$role")
done
```

Then two standard libpq features do the rest. A system-wide service file, `/etc/postgresql-common/pg_service.conf`, defines `dev_ro` and `dev_rw` as host, port, database and user. And `~/.pgpass` holds the passwords. That file is mine, so the boot only replaces its two `localhost:5432` lines and leaves anything else in it alone.

The result is that neither I nor an agent ever sees a password. `psql service=dev_ro` just works, and so does `pg_dump`.

For tools that don't read `~/.pgpass`, like Flyway or a JDBC URL, the password is read at run time rather than written into a repo:

```bash
FLYWAY_URL=jdbc:postgresql://localhost:5432/dev FLYWAY_USER=dev_rw \
FLYWAY_PASSWORD="$(awk -F: '$4=="dev_rw"{print $5}' ~/.pgpass)" flyway migrate
```

## The client has to match

Ubuntu 24.04 ships the Postgres 16 client. That can't `pg_dump` an 18 server. So the VM installs the client from the PostgreSQL project's own apt repo, with the major version taken from the same pin as the server:

```bash
apt-get install -y --no-install-recommends "postgresql-client-${PARAM_POSTGRES_VERSION%%.*}"
```

One pin, so the server and client can't drift apart. Renovate bumps that pin, and holds major versions for approval, which matters here: a Postgres major can't open an older major's data directory, so a major bump means a rebuild or a dump and restore.

## Checking the read-only role is read-only

A Lima probe runs after every boot. It waits for the database, then makes sure `dev_ro` can't write:

```bash
timeout 300s bash -c 'until psql service=dev_ro -tAc "select 1" | grep -q 1; do sleep 5; done'
# The read-only role must stay read-only. An `if`, not `!`: set -e
# ignores a negated command, so `! psql ...` could never fail the probe.
if psql service=dev_ro -c "create table devvm_probe ()" 2>/dev/null; then exit 1; fi
```

That comment is about a real bash trap. Under `set -e`, a command negated with `!` never stops the script, whatever its result. So `! psql ... create table` looks like it checks the write fails, but the probe would pass either way. The `if` actually fails it.

## Telling the agent

Claude only uses the database if it knows it's there. A repo that has data in it gets a line like this in its `CLAUDE.md`:

> Local Postgres: explore with `psql service=dev_ro` (read-only). Apply migrations and seed data with `psql service=dev_rw`, only when the task calls for it. This project's database is `<name>`.

"Only when the task calls for it" does the steering. Exploring is the default, and the role makes that safe. Changing needs a reason.

## A few limits

- **This is local data only.** Never point these service names, or anything like them, at production.
- **The data survives restarts, but not a rebuild.** It's in a Docker volume inside the VM. Anything worth keeping belongs in migrations and seed scripts in a repo.
- **To free memory**, `docker stop postgres`. It stays stopped until the next boot or `docker start postgres`.
- **Superuser access**, when it's really needed: `docker exec -it postgres psql -U postgres`.
