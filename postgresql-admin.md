# PostgreSQL Administration Cheatsheet

Ubuntu + PGDG + `postgresql-common` + systemd.

Examples use PostgreSQL 18 and cluster `main`. Replace these as needed:

```bash
export PGVER='18'
export PGCLUSTER='main'
export PGUNIT="postgresql@${PGVER}-${PGCLUSTER}.service"
```

## Table of Contents

- [Mental model](#mental-model)
- [Inspect the installation](#inspect-the-installation)
- [Create a cluster](#create-a-cluster)
- [Cluster paths](#cluster-paths)
- [Start, stop, restart, reload](#start-stop-restart-reload)
- [Configuration management](#configuration-management)
- [Connect as the bootstrap administrator](#connect-as-the-bootstrap-administrator)
- [Set up a Unix user with PostgreSQL](#set-up-a-unix-user-with-postgresql)
- [Roles](#roles)
- [Passwords](#passwords)
- [Databases](#databases)
- [Schemas](#schemas)
- [Recommended application role model](#recommended-application-role-model)
- [Existing-object privileges](#existing-object-privileges)
- [Default privileges for future objects](#default-privileges-for-future-objects)
- [Role and object inspection](#role-and-object-inspection)
- [`pg_hba.conf`](#pg_hbaconf)
- [Remote access](#remote-access)
- [Per-role defaults](#per-role-defaults)
- [Rename and remove roles](#rename-and-remove-roles)
- [Drop a database](#drop-a-database)
- [Drop schemas and clusters](#drop-schemas-and-clusters)
- [Major-version upgrades](#major-version-upgrades)
- [Useful emergency commands](#useful-emergency-commands)
- [Recommended defaults](#recommended-defaults)

---

## Mental model

```text
Ubuntu server
└── PostgreSQL major version
    └── cluster: one postgres server process
        ├── cluster-wide roles
        ├── database
        │   └── schema
        │       ├── tables
        │       ├── sequences
        │       ├── functions
        │       └── types
        └── another database
```

A Debian/Ubuntu **cluster** is identified by:

```text
<major-version>/<cluster-name>
```

For example:

```text
18/main
18/testing
17/main
```

Roles exist cluster-wide. Databases and schemas do not.

---

## Inspect the installation

```bash
psql --version
ls -1 /usr/lib/postgresql
dpkg -l 'postgresql*'
pg_lsclusters --start-conf
```

Typical output:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

The umbrella service may legitimately show `active (exited)`:

```bash
systemctl status postgresql.service
```

Inspect the actual cluster process instead:

```bash
systemctl status "$PGUNIT"
journalctl -u "$PGUNIT" -n 100 --no-pager
```

---

## Create a cluster

PGDG normally creates `main` when the first server package is installed. Check before creating another one:

```bash
pg_lsclusters
```

Create and start a cluster:

```bash
sudo pg_createcluster "$PGVER" "$PGCLUSTER" --start
```

Create on a specific port:

```bash
sudo pg_createcluster "$PGVER" "$PGCLUSTER" \
  --port '5433' \
  --start
```

Do not use dashes in cluster names because they conflict with systemd instance naming.

---

## Cluster paths

```text
Configuration:
  /etc/postgresql/<version>/<cluster>/

Main configuration:
  /etc/postgresql/<version>/<cluster>/postgresql.conf

Authentication:
  /etc/postgresql/<version>/<cluster>/pg_hba.conf

Peer/ident mappings:
  /etc/postgresql/<version>/<cluster>/pg_ident.conf

Startup policy:
  /etc/postgresql/<version>/<cluster>/start.conf

Data:
  /var/lib/postgresql/<version>/<cluster>/

Logs:
  /var/log/postgresql/postgresql-<version>-<cluster>.log

Unix sockets:
  /var/run/postgresql/
```

Ask the running server rather than assuming:

```sql
SHOW config_file;
SHOW hba_file;
SHOW ident_file;
SHOW data_directory;
SHOW port;
SHOW unix_socket_directories;
```

---

## Start, stop, restart, reload

Using the Debian/Ubuntu wrapper:

```bash
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" status
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" start
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" stop
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" restart
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" reload
```

Using systemd:

```bash
sudo systemctl start "$PGUNIT"
sudo systemctl stop "$PGUNIT"
sudo systemctl restart "$PGUNIT"
sudo systemctl reload "$PGUNIT"
```

Enable the umbrella service at boot:

```bash
sudo systemctl enable postgresql.service
```

When run as root, `pg_ctlcluster` normally redirects operations through systemd.

### `start.conf`

Edit:

```bash
sudoedit "/etc/postgresql/${PGVER}/${PGCLUSTER}/start.conf"
sudo systemctl daemon-reload
```

Values:

```text
auto
```

Starts and stops with `postgresql.service`.

```text
manual
```

Does not start automatically, but direct cluster control is allowed.

```text
disabled
```

Blocks normal `pg_ctlcluster` and systemd cluster operations. Intended to prevent accidents during maintenance.

---

## Configuration management

Edit with `vi`:

```bash
sudo pg_conftool "$PGVER" "$PGCLUSTER" edit
```

Inspect settings stored in `postgresql.conf`:

```bash
sudo pg_conftool "$PGVER" "$PGCLUSTER" show all
sudo pg_conftool "$PGVER" "$PGCLUSTER" show shared_buffers
```

Set or remove a setting:

```bash
sudo pg_conftool "$PGVER" "$PGCLUSTER" set shared_buffers '2GB'
sudo pg_conftool "$PGVER" "$PGCLUSTER" remove shared_buffers
```

Reload reloadable settings:

```bash
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" reload
```

Restart for startup-only settings:

```bash
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" restart
```

Inspect pending restart requirements:

```sql
SELECT name
     , setting
     , pending_restart
FROM pg_settings
WHERE pending_restart
ORDER BY name;
```

Check configuration files for errors:

```sql
SELECT sourcefile
     , sourceline
     , name
     , setting
     , error
FROM pg_file_settings
WHERE error IS NOT NULL;
```

---

## Connect as the bootstrap administrator

The package-created OS user and PostgreSQL superuser are normally both named `postgres`.

```bash
sudo -u postgres psql -X --dbname='postgres'
```

Useful options:

```bash
psql -X                  # Ignore ~/.psqlrc
psql -v ON_ERROR_STOP=1  # Stop scripts after the first SQL error
psql -d database
psql -U role
psql -h host
psql -p port
```

Do not use the `postgres` Unix account as an ordinary interactive account.

---

## Set up a Unix user with PostgreSQL

Local connections normally use `peer` authentication:

```text
Unix user nick -> PostgreSQL role nick
```

`psql` defaults the database role to the current Unix username. It then defaults the database name to the database role name.

This means plain `psql` works when all three names match:

```text
Unix user:       nick
PostgreSQL role: nick
Database:        nick
```

### Normal local user

```bash
sudo adduser nick

sudo -u postgres createuser \
  --no-superuser \
  --no-createdb \
  --no-createrole \
  'nick'

sudo -u postgres createdb \
  --owner='nick' \
  'nick'
```

Test:

```bash
sudo -iu nick
psql
```

Equivalent SQL:

```sql
CREATE ROLE nick
  LOGIN
  NOSUPERUSER
  NOCREATEDB
  NOCREATEROLE;

CREATE DATABASE nick
  OWNER nick;
```

Do **not** add ordinary users to the Unix `postgres` group.

### Trusted PostgreSQL administrator

Create a matching local superuser:

```bash
sudo adduser nick
sudo -u postgres createuser --superuser 'nick'
sudo -u postgres createdb --owner='nick' 'nick'
```

Or promote an existing role:

```sql
ALTER ROLE nick SUPERUSER;
```

Then:

```bash
sudo -iu nick
psql
```

Only trusted database administrators should be superusers.

### User without a same-named database

Specify a database explicitly:

```bash
sudo -iu nick psql --dbname='postgres'
```

---

## Roles

PostgreSQL has roles, not separate users and groups.

A role with `LOGIN` is a database user:

```sql
CREATE ROLE alice LOGIN;
```

A role without `LOGIN` is normally used as a group or owner:

```sql
CREATE ROLE developers NOLOGIN;
CREATE ROLE myapp_owner NOLOGIN;
```

Membership:

```sql
GRANT developers TO alice;
REVOKE developers FROM alice;
```

Prevent automatic privilege inheritance:

```sql
ALTER ROLE alice NOINHERIT;
```

The user can then explicitly assume a granted role:

```sql
SET ROLE developers;
RESET ROLE;
```

Common attributes:

```sql
ALTER ROLE alice LOGIN;
ALTER ROLE alice NOLOGIN;

ALTER ROLE alice CREATEDB;
ALTER ROLE alice NOCREATEDB;

ALTER ROLE alice CREATEROLE;
ALTER ROLE alice NOCREATEROLE;

ALTER ROLE alice SUPERUSER;
ALTER ROLE alice NOSUPERUSER;

ALTER ROLE alice CONNECTION LIMIT 20;
```

`CREATEDB`, `CREATEROLE`, `SUPERUSER`, `REPLICATION`, and `BYPASSRLS` are not inherited as ordinary object privileges.

---

## Passwords

Unix `peer` authentication does not use PostgreSQL passwords.

For TCP/password authentication, set passwords interactively:

```bash
sudo -u postgres psql --dbname='postgres'
```

```text
\password alice
```

This avoids placing the password in shell history or SQL logs.

Remove a password:

```sql
ALTER ROLE alice PASSWORD NULL;
```

Inspect the configured password format:

```sql
SHOW password_encryption;
```

Use SCRAM for new deployments:

```text
password_encryption = 'scram-sha-256'
```

`md5` password storage is deprecated.

---

## Databases

Create:

```sql
CREATE DATABASE myapp
  OWNER myapp_owner
  TEMPLATE template0
  ENCODING 'UTF8';
```

CLI equivalent:

```bash
sudo -u postgres createdb \
  --owner='myapp_owner' \
  --template='template0' \
  --encoding='UTF8' \
  'myapp'
```

Rename while connected to another database:

```sql
ALTER DATABASE myapp RENAME TO myapp_old;
```

Change owner:

```sql
ALTER DATABASE myapp OWNER TO myapp_owner;
```

Restrict public access:

```sql
REVOKE ALL ON DATABASE myapp FROM PUBLIC;
GRANT CONNECT ON DATABASE myapp TO myapp_rw;
GRANT CONNECT ON DATABASE myapp TO myapp_ro;
```

Database privileges:

```sql
GRANT CONNECT ON DATABASE myapp TO alice;
GRANT CREATE ON DATABASE myapp TO alice;
GRANT TEMPORARY ON DATABASE myapp TO alice;
```

---

## Schemas

Connect to the target database first:

```text
\connect myapp
```

Create:

```sql
CREATE SCHEMA app AUTHORIZATION myapp_owner;
```

Change owner:

```sql
ALTER SCHEMA app OWNER TO myapp_owner;
```

Grant access:

```sql
GRANT USAGE ON SCHEMA app TO myapp_ro;
GRANT USAGE ON SCHEMA app TO myapp_rw;

GRANT CREATE ON SCHEMA app TO developers;
```

`USAGE` permits resolving objects inside the schema.

`CREATE` permits creating objects inside the schema.

Harden the default `public` schema:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

Set a role-specific search path:

```sql
ALTER ROLE alice
  IN DATABASE myapp
  SET search_path = app, pg_catalog;
```

Only put trusted schemas in `search_path`.

---

## Recommended application role model

Do not let runtime login roles own application objects.

```sql
CREATE ROLE myapp_owner NOLOGIN;
CREATE ROLE myapp_ro NOLOGIN;
CREATE ROLE myapp_rw NOLOGIN;

CREATE ROLE myapp_migrate LOGIN NOINHERIT;
CREATE ROLE myapp_runtime LOGIN INHERIT;

GRANT myapp_owner TO myapp_migrate;
GRANT myapp_rw TO myapp_runtime;

CREATE DATABASE myapp OWNER myapp_owner;
```

Configure database access:

```sql
REVOKE ALL ON DATABASE myapp FROM PUBLIC;

GRANT CONNECT ON DATABASE myapp TO myapp_migrate;
GRANT CONNECT ON DATABASE myapp TO myapp_ro;
GRANT CONNECT ON DATABASE myapp TO myapp_rw;
```

Inside `myapp`:

```sql
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

CREATE SCHEMA app AUTHORIZATION myapp_owner;

GRANT USAGE ON SCHEMA app TO myapp_ro;
GRANT USAGE ON SCHEMA app TO myapp_rw;
```

The migration login should explicitly become the owner before creating objects:

```sql
SET ROLE myapp_owner;
SET search_path = app, pg_catalog;
```

This ensures new objects are owned by `myapp_owner`, not `myapp_migrate`.

---

## Existing-object privileges

Read-only:

```sql
GRANT SELECT
ON ALL TABLES IN SCHEMA app
TO myapp_ro;

GRANT SELECT
ON ALL SEQUENCES IN SCHEMA app
TO myapp_ro;
```

Read/write:

```sql
GRANT SELECT
    , INSERT
    , UPDATE
    , DELETE
ON ALL TABLES IN SCHEMA app
TO myapp_rw;

GRANT USAGE
    , SELECT
ON ALL SEQUENCES IN SCHEMA app
TO myapp_rw;
```

Functions are executable by `PUBLIC` by default. Harden them and grant explicitly:

```sql
REVOKE EXECUTE
ON ALL FUNCTIONS IN SCHEMA app
FROM PUBLIC;

GRANT EXECUTE
ON FUNCTION app.some_function(integer)
TO myapp_rw;
```

---

## Default privileges for future objects

Default privileges apply only to objects created by the specified creator role.

Tables:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE myapp_owner
IN SCHEMA app
GRANT SELECT
ON TABLES
TO myapp_ro;

ALTER DEFAULT PRIVILEGES
FOR ROLE myapp_owner
IN SCHEMA app
GRANT SELECT
    , INSERT
    , UPDATE
    , DELETE
ON TABLES
TO myapp_rw;
```

Sequences:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE myapp_owner
IN SCHEMA app
GRANT SELECT
ON SEQUENCES
TO myapp_ro;

ALTER DEFAULT PRIVILEGES
FOR ROLE myapp_owner
IN SCHEMA app
GRANT USAGE
    , SELECT
ON SEQUENCES
TO myapp_rw;
```

Remove default public function execution globally for objects created by the owner:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE myapp_owner
REVOKE EXECUTE
ON FUNCTIONS
FROM PUBLIC;
```

Important:

```text
Defaults for myapp_owner only affect objects created while current_user
is myapp_owner.
```

Membership alone is insufficient. Migration sessions should use:

```sql
SET ROLE myapp_owner;
```

Inspect default privileges:

```text
\ddp
```

---

## Role and object inspection

psql commands:

```text
\du+
\l+
\dn+
\dp app.*
\ddp
\dx
\conninfo
```

Roles:

```sql
SELECT rolname
     , rolcanlogin
     , rolsuper
     , rolcreatedb
     , rolcreaterole
     , rolinherit
     , rolconnlimit
FROM pg_roles
ORDER BY rolname;
```

Current identity:

```sql
SELECT session_user
     , current_user
     , current_role;
```

Object ownership:

```sql
SELECT n.nspname AS schema_name
     , c.relname AS object_name
     , c.relkind
     , pg_get_userbyid(c.relowner) AS owner
FROM pg_class AS c
JOIN pg_namespace AS n
  ON n.oid = c.relnamespace
WHERE n.nspname = 'app'
ORDER BY c.relkind
       , c.relname;
```

Database sessions:

```sql
SELECT pid
     , usename
     , datname
     , application_name
     , client_addr
     , state
     , query_start
     , query
FROM pg_stat_activity
ORDER BY query_start NULLS LAST;
```

---

## `pg_hba.conf`

Rules are processed top-to-bottom. The first matching rule wins.

Typical local peer rule:

```text
local   all      all                              peer
```

Local TCP with SCRAM:

```text
host    all      all      127.0.0.1/32            scram-sha-256
host    all      all      ::1/128                 scram-sha-256
```

Restricted application network:

```text
hostssl myapp    myapp_runtime 10.20.0.0/16       scram-sha-256
```

`hostssl` requires working PostgreSQL TLS configuration.

Edit:

```bash
sudoedit "/etc/postgresql/${PGVER}/${PGCLUSTER}/pg_hba.conf"
```

Validate parsed rules:

```sql
SELECT line_number
     , type
     , database
     , user_name
     , address
     , auth_method
     , error
FROM pg_hba_file_rules
ORDER BY line_number;
```

Reload:

```bash
sudo pg_ctlcluster "$PGVER" "$PGCLUSTER" reload
```

Avoid `trust` except in tightly controlled temporary environments.

---

## Remote access

Set an appropriate listen address:

```text
listen_addresses = '10.20.0.10'
```

Or all interfaces:

```text
listen_addresses = '*'
```

`listen_addresses` controls listening, not authorization. `pg_hba.conf`, TLS, and the firewall still matter.

Ubuntu firewall example:

```bash
sudo ufw allow \
  from '10.20.0.0/16' \
  to any port '5432' \
  proto tcp
```

Do not expose PostgreSQL directly to the public internet without a deliberate access architecture.

---

## Per-role defaults

```sql
ALTER ROLE myapp_runtime
  IN DATABASE myapp
  SET search_path = app, pg_catalog;

ALTER ROLE myapp_runtime
  IN DATABASE myapp
  SET statement_timeout = '30s';

ALTER ROLE myapp_runtime
  IN DATABASE myapp
  SET lock_timeout = '5s';

ALTER ROLE myapp_runtime
  IN DATABASE myapp
  SET idle_in_transaction_session_timeout = '60s';
```

Reset:

```sql
ALTER ROLE myapp_runtime
  IN DATABASE myapp
  RESET statement_timeout;

ALTER ROLE myapp_runtime RESET ALL;
```

---

## Rename and remove roles

Rename:

```sql
ALTER ROLE old_name RENAME TO new_name;
```

Before dropping a role, process every database containing objects or grants owned by it:

```sql
REASSIGN OWNED BY old_role TO new_owner;
DROP OWNED BY old_role;
```

Then drop the cluster-wide role:

```sql
DROP ROLE old_role;
```

`REASSIGN OWNED` transfers ownership.

`DROP OWNED` removes remaining grants and drops anything still owned by the role. Run it only after checking what it will affect.

---

## Drop a database

Connect somewhere else:

```text
\connect postgres
```

Current PostgreSQL:

```sql
DROP DATABASE myapp WITH (FORCE);
```

Manual session cleanup:

```sql
ALTER DATABASE myapp CONNECTION LIMIT 0;

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'myapp'
  AND pid <> pg_backend_pid();

DROP DATABASE myapp;
```

CLI:

```bash
sudo -u postgres dropdb --force 'myapp'
```

---

## Drop schemas and clusters

Drop an empty schema:

```sql
DROP SCHEMA app;
```

Drop a schema and everything inside it:

```sql
DROP SCHEMA app CASCADE;
```

Completely destroy a PostgreSQL cluster:

```bash
sudo pg_dropcluster --stop "$PGVER" "$PGCLUSTER"
```

That removes its databases, data directory, WAL, configuration, and logs.

---

## Major-version upgrades

Example: PostgreSQL 17 to 18.

Install the new server and matching extension packages first.

Check:

```bash
sudo pg_upgradecluster \
  --check \
  -v '18' \
  '17' \
  'main'
```

Upgrade using `pg_upgrade`:

```bash
sudo pg_upgradecluster \
  --method='upgrade' \
  -v '18' \
  '17' \
  'main'
```

Inspect both clusters:

```bash
pg_lsclusters --start-conf
```

Verify the new cluster before deleting the old one:

```bash
sudo -u postgres psql \
  --cluster='18/main' \
  --dbname='postgres' \
  --command='SELECT version();'
```

Remove the old cluster only after validation and backups:

```bash
sudo pg_dropcluster --stop '17' 'main'
```

`pg_upgradecluster` normally moves the old cluster to another port, marks it for manual startup, and leaves it in place for rollback.

---

## Useful emergency commands

Readiness:

```bash
pg_isready
pg_isready --host='127.0.0.1' --port='5432'
```

Server version:

```sql
SELECT version();
SHOW server_version;
SHOW server_version_num;
```

Terminate one session:

```sql
SELECT pg_terminate_backend(12345);
```

Cancel its active query without disconnecting it:

```sql
SELECT pg_cancel_backend(12345);
```

Reload from SQL:

```sql
SELECT pg_reload_conf();
```

Find blocking sessions:

```sql
SELECT blocked.pid AS blocked_pid
     , blocking.pid AS blocking_pid
     , blocked.query AS blocked_query
     , blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_locks AS blocked_lock
  ON blocked_lock.pid = blocked.pid
 AND NOT blocked_lock.granted
JOIN pg_locks AS blocking_lock
  ON blocking_lock.locktype = blocked_lock.locktype
 AND blocking_lock.database IS NOT DISTINCT FROM blocked_lock.database
 AND blocking_lock.relation IS NOT DISTINCT FROM blocked_lock.relation
 AND blocking_lock.page IS NOT DISTINCT FROM blocked_lock.page
 AND blocking_lock.tuple IS NOT DISTINCT FROM blocked_lock.tuple
 AND blocking_lock.virtualxid IS NOT DISTINCT FROM blocked_lock.virtualxid
 AND blocking_lock.transactionid IS NOT DISTINCT FROM blocked_lock.transactionid
 AND blocking_lock.classid IS NOT DISTINCT FROM blocked_lock.classid
 AND blocking_lock.objid IS NOT DISTINCT FROM blocked_lock.objid
 AND blocking_lock.objsubid IS NOT DISTINCT FROM blocked_lock.objsubid
 AND blocking_lock.pid <> blocked_lock.pid
 AND blocking_lock.granted
JOIN pg_stat_activity AS blocking
  ON blocking.pid = blocking_lock.pid;
```

---

## Recommended defaults

```text
Local human users:
  Unix account + matching PostgreSQL LOGIN role + peer authentication

Application ownership:
  NOLOGIN owner role

Migrations:
  Separate LOGIN role that explicitly SET ROLEs to the owner

Runtime:
  Separate LOGIN role inheriting a NOLOGIN privilege role

Schemas:
  Dedicated application schema
  Revoke public schema CREATE

Passwords:
  SCRAM-SHA-256

Remote access:
  Restricted address range
  TLS where appropriate
  Firewall restrictions

Superuser:
  Reserved for trusted administration
```
