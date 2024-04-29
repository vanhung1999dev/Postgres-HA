## Create a new network so our instances can talk with each other:
```
docker network create postgres
```

#### Start with instance one
```
docker run -it --rm --network postgres --name postgres-1 \
-e POSTGRES_USER=postgresadmin \
-e POSTGRES_PASSWORD=admin123 \
-e POSTGRES_DB=postgresdb \
-e PGDATA="/data" \
-v ${PWD}/postgres-1/pgdata:/data \
-v ${PWD}/postgres-1/config:/config \
-v ${PWD}/postgres-1/archive:/mnt/server/archive \
-p 5000:5432 \
postgres:15.0 -c "config_file=/config/postgresql.conf"
```

## Create a user with replica permission
```
docker exec -it postgres-1 bash

createuser -U postgresadmin -P -c 5 --replication replicationUser

```


## Update pg_hba.conf in Postgres
```
host     replication     replicationUser         0.0.0.0/0        md5
```

## Enable Write-Ahead Log and Replication
```
wal_level = replica
max_wal_senders = 3
```

## Enable Archive
```
archive_mode = on
archive_command = 'test ! -f /mnt/server/archive/%f && cp %p /mnt/server/archive/%f'

```

## Note: Need to check permission of volumn and file host

## Take a base backup
- The utility is in the PostgreSQL docker image, so let's run it without running a database as all we need is the pg_basebackup utility.
```
cd storage/databases/postgresql/3-replication

docker run -it --rm \
--net postgres \
-v ${PWD}/postgres-2/pgdata:/data \
--entrypoint /bin/bash postgres:15.0
```

- Take the backup by logging into postgres-1 with our replicationUser and writing the backup to /data.
```
pg_basebackup -h postgres-1 -p 5432 -U replicationUser -D /data/ -Fp -Xs -R
```

## Start standby instance
```
docker run -it --rm --name postgres-2 \
--net postgres \
-e POSTGRES_USER=postgresadmin \
-e POSTGRES_PASSWORD=admin123 \
-e POSTGRES_DB=postgresdb \
-e PGDATA="/data" \
-v ${PWD}/postgres-2/pgdata:/data \
-v ${PWD}/postgres-2/config:/config \
-v ${PWD}/postgres-2/archive:/mnt/server/archive \
-p 5001:5432 \
postgres:15.0 -c 'config_file=/config/postgresql.conf'
```


## Test the replication
-Let's test our replication by creating a new table in postgres-1
- On our primary instance, lets do that:
```
# login to postgres
psql --username=postgresadmin postgresdb

#create a table
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

#show the table
\dt
```
- Now lets log into our postgres-2 instance and view the table:
```
docker exec -it postgres-2 bash

# login to postgres
psql --username=postgresadmin postgresdb

#show the tables
\dt
```

## Failover
- Now lets say postgres-1 fails.
- PostgreSQL does not have built-in automated failver and recovery and requires tooling to perform this.

- When postgres-1 fails, we would use a utility called pg_ctl to promote our stand-by server to a new primary server.

- Then we have to build a new stand-by server just like we did in this guide.
- We would also need to configure replication on the new primary, the same way we did in this guide.

- Let's stop the primary server to simulate failure:
```
docker rm -f postgres-1
```

- Then log into postgres-2 and promote it to primary:
```
docker exec -it postgres-2 bash

psql --username=postgresadmin postgresdb

- confirm we cannot create a table as its a stand-by server
CREATE TABLE customers (firstname text, customer_id serial, date_created timestamp);

- exit container and run pg_ctl as postgres user (cannot be run as root!)
runuser -u postgres -- pg_ctl promote

- confirm we can create a table as its a primary server
```