# 01 · PostgreSQL

**Server:** fdfs-data  **Port:** 5432

PostgreSQL stores all FDFS data. Each service owns its own table:

| Table | Owner |
|---|---|
| `movies` | movie-service |
| `bookings` | booking-service |
| `notifications` | notification-service |

The tables are created later, from each service's own `db/schema.sql`. Here we only install the database server and create the database and user.

Run everything as root:

```shell
sudo -i
```

## 1. Install PostgreSQL

Ubuntu 24.04 ships PostgreSQL 16. Add the PostgreSQL Apt repository and install 18.

```shell
apt update
apt install -y postgresql-common ca-certificates
/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -y
apt install -y postgresql-18
```

## 2. Listen on the network

By default PostgreSQL only listens on `localhost`. The services run on another server, so it must listen on all interfaces.

```shell
sed -i "s/^#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/18/main/postgresql.conf
grep ^listen_addresses /etc/postgresql/18/main/postgresql.conf
```

## 3. Allow the app server to log in

`pg_hba.conf` decides who may connect. Allow only the `fdfs` user, to the `fdfs` database, from the app server.

```shell
echo "host    fdfs    fdfs    <APP-SERVER-IP>/32    scram-sha-256" >> /etc/postgresql/18/main/pg_hba.conf
```

> Replace `<APP-SERVER-IP>` with the private IP of fdfs-app.
> The security group allows the network path; `pg_hba.conf` decides who may log in. You need both.

## 4. Restart and enable

```shell
systemctl restart postgresql
systemctl enable postgresql
ss -lntp | grep 5432
```

You should see PostgreSQL listening on `0.0.0.0:5432`.

## 5. Create the database and user

```shell
sudo -u postgres psql -c "CREATE USER fdfs WITH PASSWORD 'FdfsPass123';"
sudo -u postgres psql -c "CREATE DATABASE fdfs OWNER fdfs;"
```

## 6. Check

```shell
sudo -u postgres psql -c "\l fdfs"
```

The tables will appear after you install the services.
