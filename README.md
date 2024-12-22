# Pevensie Postgres Driver

[![Package Version](https://img.shields.io/hexpm/v/pevensie_postgres)](https://hex.pm/packages/pevensie_postgres)
[![Hex Docs](https://img.shields.io/badge/hex-docs-ffaff3)](https://hexdocs.pm/pevensie_postgres/)

The official PostgreSQL driver for Pevensie. It provides driver
implementations for Pevensie modules to be used with Postgres
databases.

Currently provides drivers for:

- [Pevensie Auth](https://hexdocs.pm/pevensie/pevensie/auth.html)
- [Pevensie Cache](https://hexdocs.pm/pevensie/pevensie/cache.html)

## Getting Started

Configure your driver to connect to your database using the
[`PostgresConfig`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#PostgresConfig)
type. You can use the [`default_config`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#default_config)
function to get a default configuration for connecting to a local
Postgres database with sensible concurrency defaults.

```gleam
import pevensie/postgres.{type PostgresConfig}

pub fn main() {
  let config = PostgresConfig(
    ..postgres.default_config(),
    host: "db.pevensie.dev",
    database: "my_database",
  )
  // ...
}
```

Create a new driver using one of the `new_<driver>_driver` functions
provided by this module. You can then use the driver with Pevensie
modules.

```gleam
import pevensie/postgres.{type PostgresConfig}
import pevensie/auth.{type PevensieAuth}

pub fn main() {
  let config = PostgresConfig(
    ..postgres.default_config(),
    host: "db.pevensie.dev",
    database: "my_database",
  )
  let driver = postgres.new_auth_driver(config)
  let pevensie_auth = auth.new(
    driver:,
    user_metadata_decoder:,
    user_metadata_encoder:,
    cookie_key: "super secret signing key",
  )
  // ...
}
```

## Connection Management

When called with the Postgres driver, the [`connect`](https://hexdocs.pm/pevensie/pevensie/auth.html#connect) function
provided by Pevensie Auth will create a connection pool for the database. This can be called
once on boot, and will be reused for the lifetime of the application.

The [`disconnect`](https://hexdocs.pm/pevensie/pevensie/auth.html#disconnect) function will close the connection pool.

## Tables

This driver creates tables in the `pevensie` schema to store user
data, sessions, and cache data.

The current tables created by this driver are:

| Table Name | Description |
| ---------- | ----------- |
| `cache` | Stores cache data. This table is unlogged, so data will be lost when the database stops. |
| `one_time_token` | Stores one time tokens. Uses soft deletions. |
| `session` | Stores session data. |
| `user` | Stores user data. Uses soft deletions. |
| `module_version` | Stores the current versions of the tables required by this driver. Versions are stored as dates. |

### Types

Generally, the Postgres driver makes a best effort to map the types
used by Pevensie to best-practice Postgres types. Generally, columns
are non-nullable unless the Gleam type is `Option(a)`. The following
types are mapped to Postgres types:

| Gleam Type | Postgres Type |
| ---------- | ------------- |
| Any resource ID | `UUID` (generated as UUIDv7) |
| `String` | `text` |
| `tempo.DateTime` (from [`gtempo`](https://hexdocs.pm/gtempo)) | `timestamptz` |
| Record types (e.g. `user_metadata`) | `jsonb` |
| `pevensie/net.IpAddr` | `inet` |

## Migrations

You can run migrations against your
database using the provided CLI:

```sh
gleam run -m pevensie/postgres migrate --addr=<connection_string> auth cache
```

The required SQL statements will be printed to the console for use with other migration
tools like [`dbmate`](https://github.com/amacneil/dbmate). Alternatively, you can
apply the migrations directly using the `--apply` flag.

## Implementation Details

This driver uses the [pog](https://github.com/lpil/pog) library for interacting
with Postgres.

All IDs are stored as UUIDs, and are generated using using a UUIDv7 implementation
made available by [Fabio Lima](https://github.com/fabiolimace) under the MIT license.
The implementation is available [here](https://gist.github.com/fabiolimace/515a0440e3e40efeb234e12644a6a346).

### Pevensie Auth

The `user` table follows the structure of the [`User`](https://hexdocs.pm/pevensie/pevensie/user.html#User)
type. The `user_metadata` column is a JSONB column, and is used to store
any custom user metadata.

It also contains a `deleted_at` column, which is used to mark users as deleted,
rather than deleting the row from the database.

Alongside the primary key, the `user` table has unique indexes on the
`email` and `phone_number` columns. These are partial, and only index
where values are provided. They also include the `deleted_at` column,
so users can sign up with the same email if a user with that email
has been deleted.

The `session` table follows the structure of the [`Session`](https://hexdocs.pm/pevensie/pevensie/session.html#Session)
type. The `user_id` column is a foreign key referencing the `user` table. If
an expired session is read, the `get_session` function will return `None`, and
delete the expired session from the database.

#### Searching for users

The `list_users` function provided by Pevensie Auth allows you to search for users
by ID, email or phone number. With the Postgres driver, the lists in you
[`UserSearchFields`](https://hexdocs.pm/pevensie/pevensie/user.html#UserSearchFields) argument are processed
using a `like any()` where clause. This means that you can search for users
by providing a list of values, and the driver will search for users with
any of those values.

You can also use Postgres `like` wildcards in your search values. For example,
if you search for users with an email ending in `@example.com`, you can use
`%@example.com` as the search value.

### Pevensie Cache

The `cache` table is an [`unlogged`](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-UNLOGGED)
table. This ensures that writes are fast, but data is lost when the database
is stopped. Do not use Pevensie Cache for data you need to keep indefinitely.

The table's primary key is a composite of the `resource_type` and `key`
columns. This allows you to store multiple values for a given key, and
retrieve them all at once.

Writes to the cache will overwrite any existing value for the given
resource type and key. This driver does not keep a version history.

If an expired value is read, the `get` function will return `None`, and
delete the expired value from the cache.

### Custom Queries

If you wish to query the `user` or `session` tables directly, this driver
provides helper utilities for doing so.

The [`user_decoder`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#user_decoder) and
[`session_decoder`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#session_decoder) functions
can be used to decode selections from the `user` and `session` tables, respectively.

The [`user_select_fields`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#user_select_fields)
and [`session_select_fields`](https://hexdocs.pm/pevensie_postgres/pevensie/postgres.html#session_select_fields)
variables contain the SQL used to select fields from the `user` and `session`
tables for use with the `user_decoder` and `session_decoder` functions.
