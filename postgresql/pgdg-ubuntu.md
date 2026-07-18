# PGDG PostgreSQL on Ubuntu — Cheat Sheet

PGDG APT packages, versioned installs, `postgresql-common`, systemd instance
units, and Debian/Ubuntu cluster tooling.

Last checked: 2026-07-16. Examples use PostgreSQL 18.

## Table of Contents

- [Practical exercise](#practical-exercise)
- [Variables](#variables)
- [Install PGDG and PostgreSQL](#install-pgdg-and-postgresql)
- [Verify](#verify)
- [Filesystem layout](#filesystem-layout)
- [Cluster and systemd management](#cluster-and-systemd-management)
- [Logs](#logs)
- [Connect with `psql`](#connect-with-psql)
- [Create an application role and database](#create-an-application-role-and-database)
- [Configuration](#configuration)
- [Remote access](#remote-access)
- [Multiple clusters](#multiple-clusters)
- [Extensions](#extensions)
- [Backups](#backups)
- [Minor updates](#minor-updates)
- [Major upgrade](#major-upgrade)
- [Troubleshooting](#troubleshooting)
- [Destructive-command check](#destructive-command-check)
- [References](#references)

---

## Practical exercise

Before installing or upgrading anything, inspect what Ubuntu and APT currently
know. These commands do not change packages:

```bash
export PG_MAJOR='18'
. /etc/os-release

printf 'Ubuntu codename: %s\n' "$VERSION_CODENAME"
apt-cache policy "postgresql-${PG_MAJOR}" "postgresql-client-${PG_MAJOR}"
command -v psql >/dev/null && psql --version
command -v pg_lsclusters >/dev/null && pg_lsclusters --start-conf
```

```text
PGDG APT source ──► versioned server/client packages ──► cluster `18/main`
                                                                  ├── systemd instance
                                                                  ├── data + config
                                                                  └── log
```

| Output | Interpretation |
| --- | --- |
| `Candidate: (none)` | APT does not currently offer that major version. |
| Candidate from `apt.postgresql.org` | The PGDG source is visible to APT. |
| `psql (PostgreSQL) 18...` | The client executable resolves to that major version. |
| A row from `pg_lsclusters` | A Debian/Ubuntu cluster already exists; note its port and status. |

Keep the observed codename, candidate version, and cluster row beside you while
following the install steps. They connect the package layer to the running
server layer and make version mismatches easier to spot.

---

## Variables

```bash
export PG_MAJOR='18'
export PG_CLUSTER='main'
export PG_PORT='5432'
export PGCLUSTER="${PG_MAJOR}/${PG_CLUSTER}"
```

Cluster identifier:

```text
18/main
```

systemd unit:

```text
postgresql@18-main.service
```

## Install PGDG and PostgreSQL

```bash
# https://wiki.postgresql.org/wiki/Apt

sudo apt update
sudo apt install -y curl ca-certificates

sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

. /etc/os-release

sudo tee /etc/apt/sources.list.d/pgdg.sources <<EOF
Types: deb deb-src
URIs: https://apt.postgresql.org/pub/repos/apt
Suites: $VERSION_CODENAME-pgdg
Architectures: $(dpkg --print-architecture)
Components: main
Signed-By: /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc
EOF

sudo apt update
```

Inspect the package candidate:

```bash
apt-cache policy "postgresql-${PG_MAJOR}"
apt-cache madison "postgresql-${PG_MAJOR}"
```

Install a specific major version:

```bash
sudo apt install -y \
  "postgresql-${PG_MAJOR}" \
  "postgresql-client-${PG_MAJOR}"
```

Optional development packages:

```bash
sudo apt install -y \
  libpq-dev \
  "postgresql-server-dev-${PG_MAJOR}"
```

Use `postgresql-18`, not the unversioned `postgresql` meta-package, when you
want to remain on one major version.

## Verify

```bash
pg_lsclusters
pg_isready --cluster "$PGCLUSTER"

systemctl status \
  "postgresql@${PG_MAJOR}-${PG_CLUSTER}" \
  --no-pager

sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -XAt \
  -d postgres \
  -c 'select version();'
```

## Filesystem layout

```text
Config:  /etc/postgresql/<major>/<cluster>/
Data:    /var/lib/postgresql/<major>/<cluster>/
Log:     /var/log/postgresql/postgresql-<major>-<cluster>.log
Binaries:/usr/lib/postgresql/<major>/bin/
Sockets: /var/run/postgresql/
```

Important files:

```text
/etc/postgresql/<major>/<cluster>/postgresql.conf
/etc/postgresql/<major>/<cluster>/pg_hba.conf
/etc/postgresql/<major>/<cluster>/pg_ident.conf
/etc/postgresql/<major>/<cluster>/start.conf
```

Ask the server for its actual paths:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c 'show config_file;' \
  -c 'show hba_file;' \
  -c 'show data_directory;' \
  -c 'show port;'
```

## Cluster and systemd management

Inventory:

```bash
pg_lsclusters
pg_lsclusters "$PG_MAJOR" "$PG_CLUSTER"
```

`postgresql-common` commands:

```bash
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" status
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" start
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" stop
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" reload
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" restart
```

Equivalent systemd commands:

```bash
sudo systemctl status  "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
sudo systemctl start   "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
sudo systemctl stop    "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
sudo systemctl reload  "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
sudo systemctl restart "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
```

`postgresql.service` is the aggregate service. Prefer the instance unit when
operating one cluster.

## Logs

```bash
sudo tail -F \
  "/var/log/postgresql/postgresql-${PG_MAJOR}-${PG_CLUSTER}.log"

sudo journalctl \
  -u "postgresql@${PG_MAJOR}-${PG_CLUSTER}" \
  -n 100 \
  --no-pager

sudo journalctl \
  -u "postgresql@${PG_MAJOR}-${PG_CLUSTER}" \
  -f
```

## Connect with `psql`

Local superuser:

```bash
sudo -u postgres psql --cluster "$PGCLUSTER" -X -d postgres
```

TCP:

```bash
psql \
  --host '127.0.0.1' \
  --port "$PG_PORT" \
  --username 'app' \
  --dbname 'app'
```

URI:

```bash
psql 'postgresql://app@127.0.0.1:5432/app'
```

Useful `psql` commands:

```text
\conninfo       Current connection
\l+             Databases
\du+            Roles
\dn+            Schemas
\dt+            Tables
\di+            Indexes
\dx             Extensions
\d+ table_name  Describe relation
\x auto         Expanded output
\timing on      Query timing
\q              Quit
```

## Create an application role and database

Avoid putting passwords in shell history:

```bash
sudo -u postgres createuser \
  --cluster "$PGCLUSTER" \
  --no-superuser \
  --no-createdb \
  --no-createrole \
  --pwprompt \
  'app'

sudo -u postgres createdb \
  --cluster "$PGCLUSTER" \
  --owner 'app' \
  'app'
```

Verify:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c '\du+ app' \
  -c '\l+ app'
```

Destructive:

```bash
sudo -u postgres dropdb --cluster "$PGCLUSTER" 'app'
sudo -u postgres dropuser --cluster "$PGCLUSTER" 'app'
```

## Configuration

Show settings from `postgresql.conf`:

```bash
sudo pg_conftool "$PG_MAJOR" "$PG_CLUSTER" show all
sudo pg_conftool "$PG_MAJOR" "$PG_CLUSTER" show 'listen_addresses'
```

Set or remove a setting:

```bash
sudo pg_conftool \
  "$PG_MAJOR" \
  "$PG_CLUSTER" \
  set 'log_min_duration_statement' '500ms'

sudo pg_conftool \
  "$PG_MAJOR" \
  "$PG_CLUSTER" \
  remove 'log_min_duration_statement'
```

Edit with `$EDITOR`:

```bash
sudo --preserve-env=EDITOR \
  pg_conftool "$PG_MAJOR" "$PG_CLUSTER" edit
```

Apply reloadable settings:

```bash
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" reload
```

Settings waiting for a restart:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c "
    select name
         , setting
    from pg_settings
    where pending_restart
    order by name;
  "
```

Configuration errors:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c "
    select sourcefile
         , sourceline
         , name
         , setting
         , error
    from pg_file_settings
    where error is not null
       or not applied
    order by sourcefile
           , sourceline;
  "
```

## Remote access

Listen on an explicit interface:

```bash
sudo pg_conftool \
  "$PG_MAJOR" \
  "$PG_CLUSTER" \
  set 'listen_addresses' 'localhost,10.0.0.10'
```

Edit HBA rules:

```bash
sudoedit "/etc/postgresql/${PG_MAJOR}/${PG_CLUSTER}/pg_hba.conf"
```

Example:

```text
host    app    app    10.0.0.0/24    scram-sha-256
```

Use `hostssl` when TLS is configured and required.

Validate HBA parsing:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c "
    select line_number
         , type
         , database
         , user_name
         , address
         , auth_method
         , error
    from pg_hba_file_rules
    order by line_number;
  "
```

`listen_addresses` requires restart; `pg_hba.conf` only requires reload:

```bash
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" restart
sudo ss -ltnp | grep ":${PG_PORT}"
```

Optional UFW rule:

```bash
sudo ufw allow \
  from '10.0.0.0/24' \
  to any \
  port "$PG_PORT" \
  proto tcp
```

## Multiple clusters

Create a cluster on the next free port:

```bash
sudo pg_createcluster "$PG_MAJOR" 'staging' --start
```

Explicit port:

```bash
sudo pg_createcluster \
  "$PG_MAJOR" \
  'staging' \
  --port '5433' \
  --start
```

Cluster names should not contain dashes because of systemd unit naming.

Delete a cluster:

```bash
sudo pg_dropcluster --stop "$PG_MAJOR" 'staging'
```

`pg_dropcluster` deletes configuration, data, WAL, tablespaces, and logs.

## Extensions

Discover version-specific packages:

```bash
apt-cache search "^postgresql-${PG_MAJOR}-"
apt-cache search "^postgresql-${PG_MAJOR}-postgis"
apt-cache search "^postgresql-${PG_MAJOR}-pgaudit"
```

Install the exact package returned by APT:

```bash
sudo apt install "postgresql-${PG_MAJOR}-<package>"
```

Enable an extension in a database:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d 'app' \
  -c 'create extension if not exists extension_name;'
```

## Backups

Logical cluster backup using `postgresql-common`:

```bash
sudo pg_backupcluster "$PG_MAJOR" "$PG_CLUSTER" dump
sudo pg_backupcluster "$PG_MAJOR" "$PG_CLUSTER" list
```

Physical base backup:

```bash
sudo pg_backupcluster "$PG_MAJOR" "$PG_CLUSTER" basebackup
```

Default backup root:

```text
/var/backups/postgresql/
```

Restore into a new cluster name:

```bash
sudo pg_restorecluster \
  "$PG_MAJOR" \
  'restoretest' \
  '/var/backups/postgresql/18-main/<timestamp>.dump'
```

Always test restores.

### One database

```bash
sudo install \
  -d \
  -o postgres \
  -g postgres \
  -m '0750' \
  '/var/backups/postgresql/manual'

sudo -u postgres pg_dump \
  --cluster "$PGCLUSTER" \
  --format 'custom' \
  --file "/var/backups/postgresql/manual/app-$(date +%F).dump" \
  'app'
```

Restore:

```bash
sudo -u postgres createdb \
  --cluster "$PGCLUSTER" \
  --owner 'app' \
  'app_restore'

sudo -u postgres pg_restore \
  --cluster "$PGCLUSTER" \
  --dbname 'app_restore' \
  --jobs "$(nproc)" \
  '/var/backups/postgresql/manual/app-YYYY-MM-DD.dump'
```

## Minor updates

Version-specific packages remain on the installed major:

```bash
sudo apt update
sudo apt upgrade
sudo systemctl restart "postgresql@${PG_MAJOR}-${PG_CLUSTER}"
```

Verify:

```bash
sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -XAt \
  -d postgres \
  -c 'select version();'
```

## Major upgrade

Example: 17 to 18.

```bash
export OLD_MAJOR='17'
export NEW_MAJOR='18'
export PG_CLUSTER='main'
```

Take and test a backup first.

Install the new server and matching extension packages:

```bash
sudo apt update
sudo apt install -y \
  "postgresql-${NEW_MAJOR}" \
  "postgresql-client-${NEW_MAJOR}"

pg_lsclusters
```

Installing the new package may create an empty `${NEW_MAJOR}/main`. Confirm it
is empty before deleting it:

```bash
sudo pg_dropcluster --stop "$NEW_MAJOR" "$PG_CLUSTER"
```

Check dependencies:

```bash
sudo pg_upgradecluster \
  --check \
  -v "$NEW_MAJOR" \
  "$OLD_MAJOR" \
  "$PG_CLUSTER"
```

Safe, slower dump/restore upgrade:

```bash
sudo pg_upgradecluster \
  -v "$NEW_MAJOR" \
  "$OLD_MAJOR" \
  "$PG_CLUSTER"
```

Faster `pg_upgrade` method:

```bash
sudo pg_upgradecluster \
  --method 'upgrade' \
  --jobs "$(nproc)" \
  -v "$NEW_MAJOR" \
  "$OLD_MAJOR" \
  "$PG_CLUSTER"
```

Verify:

```bash
pg_lsclusters

sudo -u postgres psql \
  --cluster "${NEW_MAJOR}/${PG_CLUSTER}" \
  -X \
  -d postgres \
  -c 'select version();'
```

The new cluster takes the original port. The old cluster is retained on another
port and changed to manual startup.

After validation and any required collation work:

```bash
sudo pg_dropcluster --stop "$OLD_MAJOR" "$PG_CLUSTER"

sudo apt purge \
  "postgresql-${OLD_MAJOR}" \
  "postgresql-client-${OLD_MAJOR}"

sudo apt autoremove
```

Do not remove the old cluster until rollback is no longer required.

## Troubleshooting

```bash
pg_lsclusters
sudo pg_ctlcluster "$PG_MAJOR" "$PG_CLUSTER" status
pg_isready --cluster "$PGCLUSTER"
```

Startup failure:

```bash
sudo systemctl status \
  "postgresql@${PG_MAJOR}-${PG_CLUSTER}" \
  --no-pager

sudo journalctl \
  -u "postgresql@${PG_MAJOR}-${PG_CLUSTER}" \
  -b \
  --no-pager

sudo tail -n 200 \
  "/var/log/postgresql/postgresql-${PG_MAJOR}-${PG_CLUSTER}.log"
```

Port ownership:

```bash
sudo ss -ltnp | grep ":${PG_PORT}"
```

Installed package versions:

```bash
dpkg-query \
  -W \
  -f='${Package}\t${Version}\n' \
  "postgresql-${PG_MAJOR}" \
  "postgresql-client-${PG_MAJOR}" \
  postgresql-common \
  postgresql-client-common

"/usr/lib/postgresql/${PG_MAJOR}/bin/postgres" --version
psql --version
```

Force client wrappers to use one cluster:

```bash
export PGCLUSTER="${PG_MAJOR}/${PG_CLUSTER}"
psql -X -d postgres
```

## Destructive-command check

Before `pg_dropcluster`, `dropdb`, or manually touching the data directory:

```bash
pg_lsclusters

sudo -u postgres psql \
  --cluster "$PGCLUSTER" \
  -X \
  -d postgres \
  -c 'select version();' \
  -c '\l+'

sudo pg_backupcluster "$PG_MAJOR" "$PG_CLUSTER" list
```

Prefer `pg_dropcluster` over manually deleting `/var/lib/postgresql/...`.

## References

- [PostgreSQL Ubuntu packages](https://www.postgresql.org/download/linux/ubuntu/)
- [PGDG APT repository](https://wiki.postgresql.org/wiki/Apt)
- [PGDG APT FAQ](https://wiki.postgresql.org/wiki/Apt/FAQ)
- [pg_wrapper](https://manpages.debian.org/testing/postgresql-client-common/pg_wrapper.1.en.html)
- [postgresql-common manpages](https://manpages.debian.org/testing/postgresql-common/)
