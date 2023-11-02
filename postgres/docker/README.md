# Intro to Postgres in Docker

PostgreSQL [Docker Image](https://hub.docker.com/_/postgres)

---

## Running PostgreSQL via Docker

- Docker: `docker run --name my-postgres -e POSTGRES_PASSWORD=<password> -d postgres`

- Docker-Compose (in the designated location): `docker-compose up`
- Adminer can then be accessed via `localhost:8080`, providing the required credentials (defaults for username and database = `postgres`).
  - This then allows access to Tables and Views in a Web UI. PGAdmin can also be used for this.

## Persisting Data

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

## Running Commands

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

## Network Configuration

- By default, Postgres runs on port 5432, in Docker this can be bound to something else. When using `docker run` add `-p <local port>:5432`
- Postgres could also be configured to run on a different port by default, this is handled by any of config files and/or env variables.