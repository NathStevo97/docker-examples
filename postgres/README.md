# Introduction to Postgres in Docker

PostgreSQL [Docker Image](https://hub.docker.com/_/postgres)

---

## Introduction

### Running PostgreSQL via Docker

- Docker: `docker run --name my-postgres -e POSTGRES_PASSWORD=<password> -d postgres`

- Docker-Compose (in the designated location): `docker-compose up`
- Adminer can then be accessed via `localhost:8080`, providing the required credentials (defaults for username and database = `postgres`).
  - This then allows access to Tables and Views in a Web UI. PGAdmin can also be used for this.

### Persisting Data

- One must utilise volumes to persist data in Postgres, otherwise it is deleted upon container deletion.
- Data is stored by default in `/var/lib/postgresql/data`, so mount this directory to a directory of choice in the local system.
- When running via Docker:

```shell
docker run -it --rm --name postgres `
  -e POSTGRES_PASSWORD=admin123 `
  -v ${PWD}/pgdata:/var/lib/postgresql/data `
  postgres:15.0
```

- The volume may also be mounted within Docker-compose file.

### Running Commands

```shell
# Exec into the container

docker exec -it <container name> bash

## Login to postgres

psql -h localhost -U postgres # by default if connecting from within the instance, no password required here

# Create a table

CREATE TABLE customers (firstname text,lastname text, customer_id serial);

# Add a record

INSERT INTO customers (firstname, lastname) VALUES ( 'Nathan', 'Stephenson');

# Show a table

\dt

# Get records

SELECT * FROM customers;

# Quit Postgres
\q
```

### Network Configuration

- By default, Postgres runs on port 5432, in Docker this can be bound to something else. When using `docker run` add `-p <local port>:5432`
- Postgres could also be configured to run on a different port by default, this is handled by any of config files and/or env variables.

---

## Configuration

- **Reminder** - PostgreSQL can be spun up via `docker run` in a similar manner to the following:

```shell
docker run -it --rm --name postgres `
  -e POSTGRES_PASSWORD=admin123 `
  -v ${PWD}/pgdata:/var/lib/postgresql/data `
  -p <local port>:5432
  postgres:15.0
```

- Common Environment Variables that PostgreSQL can leverage include:

| Environment Variable | Description                                                                           |
|----------------------|---------------------------------------------------------------------------------------|
| `POSTGRES_PASSWORD`  | The password for the Postgres Admin (required)                                        |
| `POSTGRES_USER`      | The username for the Postgres Admin (optional, defaults to `postgres`)                |
| `POSTGRES_DB`        | Name of the initial Postgres Database created (optional, defaults to `POSTGRES_USER`) |
| `PGDATA`             | The location for the database files to be stored.                                     |

### Configuration Files

- The directory mounted via `${PWD}/pgdata:/var/lib/postgresql/data ` is used to store not only the data for the Postgres Database, but configuration for it as well.
- Key Configuration files included are:

| Config File       | Description                      | Documentation     |
|-------------------|----------------------------------|-------------------|
| `pg_hba.conf`     | Host-Based Authentication        | [Documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html) |
| `pg_ident.conf`   | User Mappings                    | [Documentation](https://www.postgresql.org/docs/current/auth-username-maps.html) |
| `postgresql.conf` | PostgreSQL primary Configuration |                   |

- Each of the config files defaults to the container directory specified in the command above, this can be configured if desired.

### pg_hba.conf

- Controls device and user access to the PostgreSQL server.
- General format follows a table:

```shell
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust

host all all all scram-sha-256
```

- Guidance on each field is provided in the default configuration and the docuemntation.

### postgresql.conf

- The primary configuration used with PostgreSQL
- An extremely large file with lots of options.

- The location for the "data" stored by Postgres, the `hba` and `ident` files can be configured in this:

```
data_directory = '/data'
hba_file = '/config/pg_hba.conf'
ident_file = '/config/pg_ident.conf'
```

### Connection and Authentication

- `shared_buffers` - Determines how much memory is dedicated to the server for caching data; defaults to 15-25% of the machine's total RAM.
- Example network, Write-Ahead-Log (WAL), replication configuration follows:

```shell
port = 5432
listen_addresses = '*'
max_connections = 100
shared_buffers = 128MB
dynamic_shared_memory_type = posix
max_wal_size = 1GB
min_wal_size = 80MB
log_timezone = 'Etc/UTC'
datestyle = 'iso, mdy'
timezone = 'Etc/UTC'

#locale settings
lc_messages = 'en_US.utf8'			# locale for system error message
lc_monetary = 'en_US.utf8'			# locale for monetary formatting
lc_numeric = 'en_US.utf8'			# locale for number formatting
lc_time = 'en_US.utf8'				# locale for time formatting

default_text_search_config = 'pg_catalog.english'
```

- **Note:** - One can also include configuration from other locations e.g. `includ_dir`, `include`.

### Custom Configuration

- When running on Linux, the `postgres` user (with default user id `999`) must have access to these configuration files:

```shell
sudo chown 999:999 config/postgresql.conf
sudo chown 999:999 config/pg_hba.conf
sudo chown 999:999 config/pg_ident.conf
```

- **Note:** The `PGDATA` variable tells PostgreSQL where the data directory, and the `data_directory` parameter in the config file also gives this information.
  - The latter is only read by PostgreSQL after initialization.
  - Initialization sets up directory permissions on the data directory.
  - If `PGDATA` is left out; errors will occur implying the data directory is invalid.

### Running PostgreSQL

```shell
docker run -it --rm --name postgres `
-e POSTGRES_USER=postgresadmin `
-e POSTGRES_PASSWORD=admin123 `
-e POSTGRES_DB=postgresdb `
-e PGDATA="/data" `
-v ${PWD}/pgdata:/data `
-v ${PWD}/config:/config `
-p 5000:5432 `
postgres:15.0 -c 'config_file=/config/postgresql.conf'
```

---

## Replication

### Setup

- Now the postgres instances will be referred to by numeric identifiers, `postgres-1` and `postgres-2` for the primary and secondary servers respectively.
  - This can be achieved in the `docker run` command by adding `--name <name>`
- Each of the servers should have their own unique data volumes and config files, but should run within the same container network.

- Setup the docker network: `docker network create postgres`
- Initialize instance 1:

```shell
docker run -it --rm --name postgres-1 `
--net postgres `
-e POSTGRES_USER=postgresadmin `
-e POSTGRES_PASSWORD=admin123 `
-e POSTGRES_DB=postgresdb `
-e PGDATA="/data" `
-v ${PWD}/postgres-1/pgdata:/data `
-v ${PWD}/postgres-1/config:/config `
-v ${PWD}/postgres-1/archive:/mnt/server/archive `
-p 5000:5432 `
postgres:15.0 -c 'config_file=/config/postgresql.conf'
```

### Create Replication User

- A PostgreSQL user account with replication permissions is required before it can be setup:

```shell
docker exec -it postgres-1 bash

# create the user
createuser -U postgresadmin -P -c 5 --replication replicationUser

exit
```

- To allow the `replicationUser` access to the database, it must be added to `pg_hba.conf`. The following sample does this and allows connection from anywhere for the user.

```shell
host     replication    replicationUser     0.0.0.0/0    md5
```

### Write-Ahead Logging (WAL) and Replication

- WAL is one of the key features for High-Availability in Postgres.
- Essentially, this ensures [transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html#TUTORIAL-TRANSACTIONS) are logged to a file.
  - The transaction is not considered complete until it has been logged accordingly and flushed to disk.
- So, if a crash in the system occurs, the database can be recovered from the transaction log - "writing ahead".

- Additional documentation:
  - [WAL](https://www.postgresql.org/docs/current/wal-intro.html#WAL-INTRO)
  - `wal_level` [configuration](https://www.postgresql.org/docs/current/runtime-config-wal.html)
  - `max_wal_senders` [configuration](https://www.postgresql.org/docs/current/runtime-config-replication.html)

```shell
wal_level = replica # how much information is written to the WAL - replica is the default
max_wal_senders = 3 # the maximum number of concurrent connections from standby servers or streaming base backup clients.
```

### Archiving

- To ensure the transaction log can be recovered and therefore Postgres, it's strongly advised to configure archiving for Postgres.
- This ensures the transaction log is persisted to a folder.
- [Documenation](https://www.postgresql.org/docs/current/runtime-config-wal.html#GUC-ARCHIVE-MODE)

- Adding the configuration to `postgresql.conf`:

```shell
archive_mode = on
archive_command = 'test ! -f /mnt/server/archive/%f && cp %p /mnt/server/archive/%f' # check the existence of the path and copy to the archive location
```

### Taking Backups

- PostgreSQL by default doesn't come with much HA configuration, but there are tools to facilitate it such as the `pg_basebackup` [utility](https://www.postgresql.org/docs/current/app-pgbasebackup.html), which comes in the PostgreSQL docker image.
- This can be triggered without creating the database:

```shell
docker run -it --rm `
--net postgres `
-v ${PWD}/postgres-2/pgdata:/data `
--entrypoint /bin/bash postgres:15.0
```

- **Note:** the directory mounted is for `postgres-2` which is acting as the backup server.
- The backup can be triggered by logging into `postgres-1` and write it to a `data/` directory to be referenced by the `postgres-2`:

```shell
pg_basebackup -h postgres-1 -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
```

- A backup will be written to the `archive` folder, and the `pgdata` from `postgres-1` will be written to the `postgres-2`  folder.

### Starting the Standby Server

- Ensure the specific configs are passed; archival is going to be enabled on this server as well.

```shell
docker run -it --rm --name postgres-2 `
--net postgres `
-e POSTGRES_USER=postgresadmin `
-e POSTGRES_PASSWORD=admin123 `
-e POSTGRES_DB=postgresdb `
-e PGDATA="/data" `
-v ${PWD}/postgres-2/pgdata:/data `
-v ${PWD}/postgres-2/config:/config `
-v ${PWD}/postgres-2/archive:/mnt/server/archive `
-p 5001:5432 `
postgres:15.0 -c 'config_file=/config/postgresql.conf'
```

### Testing the Replication

- Login to the primary instance and perform some action e.g. create and view a table:

```shell
# login to postgres
psql --username=postgresadmin postgresdb

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt
```

- Logging into the `postgres-2` instance, the same table should be viewable:

```shell
docker exec -it postgres-2 bash

# login to postgres
psql --username=postgresadmin postgresdb

#show the tables
```

### Simulating Failover

- Suppose `postgres-1` fails, there is no autmated failover recovery to help, additional tooling is required again, in this case `pg_ctl`.
- `pg_ctl` will be used to promote the standby server to be the new primary.

- `postgres-1` failure can be simulated by stopping the container: `docker rm -f postgres-1`

- Logging into the `postgres-2` to promote it to primary and verifying non-read-only capabilities:

```shell
docker exec -it postgres-2 bash

# confirm we cannot create a table as its a stand-by server
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

# run pg_ctl as postgres user (cannot be run as root!)
runuser -u postgres -- pg_ctl promote

# confirm we can create a table as its a primary server
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);
```

---

## Kubernetes Introduction

- When running in Kubernetes, one must consider how containers are affected by cluster upgrades. This is especially important for PostgreSQL, which isn't made for Kubernetes by default, and the data it stores.
- Typically cluster upgrades do rolling node upgrades in cloud-based environments. This may lead to primary servers being deleted and restored without data.
- Cluster upgrade and deployment strategies are therefore critical, and must leverage operators and controllers accordingly.

### Getting Started

- Create a cluster via whatever means desired e.g. `k3d`, `kind`, etc. A sample command for `kind` follows: `kind create cluster --name postgresql --image kindest/node:v1.28.0`
- Create a namespace: `kubectl create ns postgresql`
- For basic use, it's advised to define the environment variables as Kubernetes secrets:

```shell
kubectl -n postgresql create secret generic postgresql `
  --from-literal POSTGRES_USER="postgresadmin" `
  --from-literal POSTGRES_PASSWORD='admin123' `
  --from-literal POSTGRES_DB="postgresdb" `
  --from-literal REPLICATION_USER="replicationuser" `
  --from-literal REPLICATION_PASSWORD='replicationPassword'
```

### PostgreSQL Deployment - StatefulSets

- To ensure data is persisted, a StatefulSet will be utilised.
- Sample configuration is available in the [documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/), this should have the following areas updated:
  - Name: `postgres`
  - Service: Use `5432` as required by PostgreSQL
  - Supply the environment variables via the secret
  - Provide the `.conf` files used for `PostgreSQL` via a ConfigMap

- Once configured, deploy via `kubectl -n postgresl create -f <file path>`

- Verify the installation via checking the pod status and logs:
  - `kubectl -n postgresql get pods`
  - `kubectl -n postgresql logs postgres-0`

- **Note:** Errors will be shown in the logs as archive has not been setup; this is fine for now.
- Exec-ing into the instance to check its capabilities:

```shell
kubectl -n postgresql exec -it postgres-0 -- bash

# login to postgres
psql --username=postgresadmin postgresdb

# see our replication user created
\du

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt

# quit out of postgresql
\q

# check the data directory
ls -l /data/pgdata

# check the archive (does not exist!)
ls -l /data/archive
```

### Enhancing with Init Containers

- To help with preamble tasks, in this case creating the `/data/archive` directory and additional configuration, `InitContainers` can be leveraged.
- These are added to the template spec of the statefulset.

```yaml
initContainers:
- name: init
  image: postgres:15.0
  command: [ "bash", "-c" ]
  args:
  - |
    #create archive directory
    mkdir -p /data/archive && chown -R 999:999 /data/archive
```

- For archival setup, volumeMounts need to be configured:

```yaml
volumeMounts:
- name: data
  mountPath: /data
  readOnly: false
```

- Redeploying via `kubectl apply` and verifying deployment as before, one can then re-exec into the container and check for data persisting:

```shell
kubectl -n postgresql exec -it postgres-0 -- bash
ls /data
ls /data/archive/

# check if our table was persisted!
psql --username=postgresadmin postgresdb
\dt
\q
```

---

## Replication in Kubernetes

- The initial steps for replication in Kubernetes require edits to the StatefulSet configuration already created.
- Additional environment variables need to be set, the two in question should be in the secret created:
  - `REPLICATION_USER="<replication username>"`
  - `REPLICATION_PASSWORD="<replication password>"`

- As with the Docker-based replication setup, a numeric naming convention should be utilised for easy instance identification.

### Initialization

- As described on the Docker Hub [documentation](https://hub.docker.com/_/postgres#:~:text=and%20POSTGRES_DB.-,Initialization%20scripts,-If%20you%20would) for Postgres, initialization scripts can be ran from `/docker-entrypoint-initdb.d/`
  - Anything in the folder above will only run when the data directory has not been initialized i.e. we only want it to run one time.
- To setup, `volumeMounts:` and `volumes:` must be setup accordingly. The volume just needs to be an empty directory to allow file sharing:

```yaml
volumeMounts:
- mountPath: /docker-entrypoint-initdb.d
  name: initdb
...
volumes:
- name: initdb
  emptyDir: {}
```

- With volume configuration complete, a script can be ran to create the replicationuser using the initcontainer, added to the `args` section of the container spec:

```yaml
args:
        - |
          if [ ${STANDBY_MODE} == "on" ];
          then
            # initialize from backup if data dir is empty
            if [ -z "$(ls -A ${PGDATA})" ]; then
              export PGPASSWORD=${REPLICATION_PASSWORD}
              pg_basebackup -h ${PRIMARY_SERVER_ADDRESS} -p 5432 -U ${REPLICATION_USER} -D ${PGDATA} -Fp -Xs -R
            fi
          else
            #create archive directory
            mkdir -p /data/archive && chown -R 999:999 /data/archive

            #create a init template
            echo "CREATE USER #REPLICATION_USER REPLICATION LOGIN ENCRYPTED PASSWORD '#REPLICATION_PASSWORD';" > init.sql

            # add credential
            sed -i 's/#REPLICATION_USER/'${REPLICATION_USER}'/g' init.sql
            sed -i 's/#REPLICATION_PASSWORD/'${REPLICATION_PASSWORD}'/g' init.sql

            mkdir -p /docker-entrypoint-initdb.d/
            cp init.sql /docker-entrypoint-initdb.d/init.sql
          fi
```

- The environment variables associated with replication need to be injected from the secret defined previously, this can be done via `valueFrom:`

### Installation

- Deployment via `kubectl apply` or `create` as appropriate. Installation can be verified by checking the logs and exec-ing into the container:

```shell
kubectl -n postgresql get pods

# check the initialization logs (should be clear)
kubectl -n postgresql logs postgres-1-0 -c init

# check the database logs
kubectl -n postgresql logs postgres-1-0

kubectl -n postgresql exec -it postgres-1-0 -- bash

# login to postgres
psql --username=postgresadmin postgresdb

# see our replication user created
\du

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt

# quit out of postgresql
\q

# check the data directory
ls -l /data/pgdata

# check the archive
ls -l /data/archive
```

### Standby Server Setup

- Similar to Docker-based replication, the standby-instance should be named `postgres-2` or appropriate and have replication streaming configured, this should be handled by the initContainer.
- To avoid massive differences between files, a new set of environment variables will be included:
  - `STANDBY_MODE` - "on" for standby, "off" for primary functionality.
  - `PRIMARY_SERVER_ADDRESS` - The container name for the primary server e.g. `"postgres-1"`
  - `PGDATA` - The location of the streaming data backup.

- The initContainer can use a similar script as the primary server, with a conditional based on the `STANDBY_MODE` value, e.g.:

```shell
if [ ${STANDBY_MODE} == "on" ];
then
  # initialize from backup if data dir is empty
  if [ -z "$(ls -A ${PGDATA})" ]; then
    export PGPASSWORD=${REPLICATION_PASSWORD}
    pg_basebackup -h ${PRIMARY_SERVER_ADDRESS} -p 5432 -U ${REPLICATION_USER} -D ${PGDATA} -Fp -Xs -R
  fi
else
  # previous logic here
fi
```

### Standby Deployment

- Deploying via `kubectl apply`, installation and replication should be verified:

```shell
kubectl -n postgresql get pods

# check the initialization logs (you'll see `No such file or directory` because its not initialized yet!)
kubectl -n postgresql logs postgres-2-0 -c init

# check the database logs
kubectl -n postgresql logs postgres-2-0

kubectl -n postgresql exec -it postgres-2-0 -- bash

# login to postgres
psql --username=postgresadmin postgresdb

# see our replication user created
\du

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt

# quit out of postgresql
\q

# check the data directory
ls -l /data/pgdata
```

### Failover

- In the event of failover, we need to use `pg_ctl` again.
- Failover can be simulated by deleting the primary statefulset: `kubectl delete sts <name>`
  - Replication failure can be noted in the pods for `postgres-2`
- Promoting `postgres-2` can be done with relative ease:

```shell
kubectl -n postgresql exec -it postgres-2-0 -- bash
psql --username=postgresadmin postgresdb

# confirm we cannot create a table as its a stand-by server
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

# quit out of postgresql
\q

# run pg_ctl as postgres user (cannot be run as root!)
runuser -u postgres -- pg_ctl promote

# confirm we can create a table as its a primary server
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);
```

- Replication will then need to be set on the new primary, turning `STANDBY_MODE` to `"off"` and `PRIMARY_SERVER_ADDRESS` to "".

- If a new standby server was to be delployed, simply copy the old configuration of `postgres-2` and ensure `PRIMARY_SERVER_ADDRESS` is configured accordingly.
