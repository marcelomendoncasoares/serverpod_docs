# serverpod database

The `serverpod database` command manages the embedded PostgreSQL database of a Serverpod project. Use `serverpod database start` to boot the database on its own, without starting the server, and keep it running until you stop it with Ctrl+C. If the server already runs the database, the command joins it instead. Either way, it prints the Unix socket and TCP addresses that external database tools connect to.

The database configuration comes from the run mode selected with `--mode`, which must have `database.dataPath` set. For that option and its default, see the [Configuration reference](../../../lookups/configuration-reference.md).
