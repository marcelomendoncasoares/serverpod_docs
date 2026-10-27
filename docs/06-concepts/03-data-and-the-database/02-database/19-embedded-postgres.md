---
description: Embedded PostgreSQL is a real PostgreSQL server that Serverpod starts and stops with your server, for local development and testing without Docker.
---

# Embedded PostgreSQL

New projects run an embedded PostgreSQL in the `development` and `test` run modes. It is a real PostgreSQL server that Serverpod starts and stops together with your server, so local development gets the same database engine as production without installing PostgreSQL or running Docker.

It is meant for local development and testing only. There are no backups, no replication, and nothing supervising the process, so staging and production connect to a managed or separately operated PostgreSQL instead. See [Connection](./connection) for those setups.

## Enable it

Embedded PostgreSQL is enabled by the `dataPath` setting, which generated projects already include:

```yaml title="config/development.yaml"
database:
  host: localhost
  port: 8090
  name: myproject
  user: postgres
  dataPath: .serverpod/development/pgdata
```

The rest of the database settings stay as they are. Serverpod uses `name` and `user` when it creates the database and reads the password from `config/passwords.yaml`, exactly as it would for an external instance. Remove `dataPath` to connect to the `host` and `port` instead.

A relative `dataPath` is resolved from the root of the server package, and each run mode needs its own directory, for example `.serverpod/test/pgdata` in `config/test.yaml`. The directory holds a complete database, so keep it out of version control.

## What happens when the server starts

The first start downloads the PostgreSQL binaries for your operating system and architecture into a per-user cache, which every later start and every other project reuses. Binaries are available for Linux and macOS on x64 and Arm64, and for Windows on x64.

The cache location depends on your operating system:

- **macOS:** `~/Library/Caches/serverpod/pg-binaries`
- **Linux:** `$XDG_CACHE_HOME/serverpod/pg-binaries`, or `~/.cache/serverpod/pg-binaries` when `XDG_CACHE_HOME` is not set
- **Windows:** `%LOCALAPPDATA%\serverpod\Cache\pg-binaries`

Set `SERVERPOD_PG_CACHE_DIR` to keep the binaries in a different directory.

Serverpod starts the database before it opens its connection pool and connects to it over a Unix domain socket. When `config/passwords.yaml` has a `database` password for the run mode, as new projects do, the database also listens on the configured `port`, so other tools can connect to it. If another Serverpod process already runs a database in the same `dataPath`, the new process attaches to it rather than starting a second one.

If another process already holds the port, such as a Docker PostgreSQL from an older project, the start fails and names the port. Under `serverpod start`, a port held by another `serverpod start`, for example in another worktree, moves the whole stack to free ports instead.

The files under `dataPath` are kept when the server stops, so your data is still there on the next start. After an unclean exit, Serverpod clears the leftover process state and brings the database back up.

## Connect a database tool

Clients like `psql` and `pgAdmin` connect to `localhost` on the configured `port`. Use the `name` and `user` from the run mode's configuration, and the password from `config/passwords.yaml`.

To start the database without the server, or to print its addresses, run:

```bash
$ serverpod database start
```

It boots the database configured for the `development` run mode, prints its Unix socket and TCP addresses, and keeps it running until you stop it with Ctrl+C. If the server already runs the database, the command joins it instead, and the database stops when the server does.

Pass `--mode` to pick another run mode and `--port` to listen somewhere else. The `--port` option needs a database password, and fails when the database already runs on another port:

```bash
$ serverpod database start --mode test --port 9090
```

## Run integration tests

Integration tests bring their own database, so there is nothing to install or start first:

```bash
$ dart test
```

The started database server will be held open until the test process exits. Every suite in a test process shares the same database server, but each `withServerpod` group creates a randomly named database of its own on it that will be dropped on teardown. That is where the isolation comes from: groups run in parallel without seeing each other's data or leaving anything behind between runs.

The data directory is not temporary. Like the one for `development`, it stays after the tests, and only the server process is reclaimed when the test process exits. See [Get started with testing](../../testing/get-started) for the rest of the setup.

:::note
If a test process is hard-killed, the database server will not be stopped and data can be left behind. This data won't harm future runs because of the named database isolation, but it can occupy disk space. If the directory happens to grow too large, it is fully safe to delete it and let the next test recreate it from scratch.
:::

## Reset the database

To start from an empty database, stop the server and delete the directory `dataPath` points at. Serverpod creates a new one on the next start.

:::warning
Deleting `dataPath` permanently deletes the database and all local data stored in it.
:::

## Related

- [Connection](./connection): connecting to a PostgreSQL instance you run yourself.
- [Configuration](../../server-fundamentals/configuration#database-backends): the database section of the run mode configuration.
- [`serverpod database`](../../cli/commands/database): every option of the command above.
- [serverpod_embedded_postgres](https://pub.dev/packages/serverpod_embedded_postgres): the package behind this, for tools that need a PostgreSQL process of their own.
