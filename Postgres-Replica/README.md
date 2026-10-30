### Immich PostgreSQL Streaming Replication Setup

This repository contains the configuration and operational guides for a real-time, read-only PostgreSQL streaming replica for an **Immich** server. This setup provides high availability, live data backups, and offloads analytical queries from the primary server. 

### Topology Overview

* **Primary Database Server:** 

  * **Host IP:** 192.168.5.45
  * **Port:** 5432
  * **Image:** ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
* **Standby Replica Server:** 

  * **Host IP:** [Your Replica IP]
  * **Port:** 5433 (Host) -> 5432 (Container)
  * **Image:** Same as Primary (Binary-compatible)

### 1. Primary Server Configuration

The primary database must have its Write-Ahead Log (WAL) level upgraded to allow replication streaming and accept remote connections. 

### Docker Compose Block (docker-compose.yml)

yaml

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    shm_size: 128mb
    restart: always
    command: >
      postgres 
      -c wal_level=replica 
      -c max_wal_senders=5 
      -c max_replication_slots=5 
      -c hot_standby=on
    healthcheck:
      disable: false

Use code with caution.

### Access Control Configuration

1. **Replication User:** Create the dedicated replication account. 

bash

docker exec -it immich_postgres psql -U postgres -c "CREATE USER replicator WITH REPLICATION PASSWORD 'YourSecureReplicaPassword';"

Use code with caution.
2. **Network Whitelisting (pg_hba.conf):** Append the following line to allow your replica to connect over the local network: 

bash

docker exec -it immich_postgres sh -c "echo 'host replication replicator 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"

Use code with caution.
3. **Reload Configuration:** Reload settings without restarting the container: 

bash

docker exec -it immich_postgres psql -U postgres -c "SELECT pg_reload_conf();"

Use code with caution.

### 2. Replica Server Configuration

The standby replica uses a bootstrap script to check for data availability. If empty, it securely clones the primary engine using pg_basebackup before starting in read-only mode. 

### Docker Compose Block (docker-compose.yml)

yaml

services:
  postgres-replica:
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:bcf63357191b76a916ae5eb93464d65c07511da41e3bf7a8416db519b40b1c23
    container_name: immich_postgres_replica
    ports:
      - "5433:5432" # Bound to 5433 on the host to avoid port clashes
    environment:
      - POSTGRES_PASSWORD=YourSecurePassword
      - PGPASSWORD=YourSecureReplicaPassword
      - PGUSER=postgres # Fixes healthcheck "role root does not exist" errors
    volumes:
      - pg_replica_data:/var/lib/postgresql/data
    entrypoint: [ "/bin/bash", "-c" ]
    command:
      - |
        if [ -z "$(ls -A /var/lib/postgresql/data)" ]; then
          echo "Data directory is empty. Initializing base backup from primary..."
          pg_basebackup -h 192.168.5.45 -D /var/lib/postgresql/data -U replicator -Fp -Xs -P -R
        fi

        # Explicitly force recovery standby state
        touch /var/lib/postgresql/data/standby.signal

        chown -R postgres:postgres /var/lib/postgresql/data
        exec docker-entrypoint.sh postgres -c hot_standby=on

volumes:
  pg_replica_data:

Use code with caution.

### 3. Operations & Maintenance

### Verify Streaming Status

To check if the replication link is healthy and actively streaming, execute this command on the **Primary Server**: 

bash

docker exec -it immich_postgres psql -U postgres -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"

Use code with caution.

* **Expected Output:** A row showing your Replica's IP with state streaming.

To verify the standby state from the **Replica Server**: 

bash

docker exec -it immich_postgres_replica psql -U postgres -c "SELECT pg_is_in_recovery();"

Use code with caution.

* **Expected Output:** t (True).

### Resyncing / Fixing System Identifier Mismatches

If the replica database loses synchronization, gets corrupted, or hits a database system identifier differs error, you must clear the mismatched local volume and initiate a clean resync: 

bash

# 1. Stop the container
docker compose down

# 2. Clear out the corrupted volume data
docker volume rm [project-name]_pg_replica_data

# 3. Re-launch the container to pull a fresh copy
docker compose up -d

Use code with caution.
