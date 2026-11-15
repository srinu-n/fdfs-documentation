# 04 · movie-service

**Server:** fdfs-app  **Port:** 8001  **Language:** Python

movie-service lists the movies and show times. It reads them from PostgreSQL and caches the list in Redis for 30 seconds.

**Talks to:** PostgreSQL (`movies` table) and Redis

Run everything as root:

```shell
sudo -i
```

## 1. Install Python

```shell
apt update
apt install -y python3 python3-venv python3-pip unzip curl postgresql-client
```

> `postgresql-client` gives us `psql`, used below to load the table schema.

## 2. Create the app user

Applications should never run as root. All FDFS services run as the system user `fdfs`, which cannot log in.

```shell
id fdfs || useradd --system --no-create-home --shell /usr/sbin/nologin fdfs
```

> `id fdfs ||` skips the command if an earlier service already created the user.

## 3. Create the service folder

```shell
mkdir -p /opt/fdfs/movie-service
```

## 4. Download the code

```shell
curl -o /tmp/movie-service.zip https://<ARTIFACTS-BUCKET>.s3.ap-south-1.amazonaws.com/movie-service-v1.zip
cd /opt/fdfs/movie-service
unzip -o /tmp/movie-service.zip
ls
```

## 5. Create a virtualenv and install packages

A virtualenv keeps this service's Python packages separate from the system and from the other services.

```shell
python3 -m venv /opt/fdfs/movie-service/venv
/opt/fdfs/movie-service/venv/bin/pip install -r /opt/fdfs/movie-service/requirements.txt
chown -R fdfs:fdfs /opt/fdfs/movie-service
```

## 6. Write the environment file

All settings come from environment variables. Nothing is hard-coded in the app.

```shell
mkdir -p /etc/fdfs
vim /etc/fdfs/movie-service.env
```

```ini
DB_HOST=<DATA-SERVER-IP>
DB_PORT=5432
DB_NAME=fdfs
DB_USER=fdfs
DB_PASSWORD=FdfsPass123
REDIS_HOST=<DATA-SERVER-IP>
REDIS_PORT=6379
```

```shell
chmod 600 /etc/fdfs/movie-service.env
```

> Replace `<DATA-SERVER-IP>` with the private IP of fdfs-data. `chmod 600` because the file holds passwords.

## 7. Load the table schema

`schema.sql` creates the `movies` table. `seed.sql` adds six sample shows (it empties the table first, so it is safe to run again).

```shell
PGPASSWORD=FdfsPass123 psql -h <DATA-SERVER-IP> -U fdfs -d fdfs -f /opt/fdfs/movie-service/db/schema.sql
PGPASSWORD=FdfsPass123 psql -h <DATA-SERVER-IP> -U fdfs -d fdfs -f /opt/fdfs/movie-service/db/seed.sql
```

## 8. Create the systemd service

```shell
vim /etc/systemd/system/fdfs-movie-service.service
```

```ini
[Unit]
Description=FDFS movie-service
After=network-online.target
Wants=network-online.target

[Service]
User=fdfs
Group=fdfs
WorkingDirectory=/opt/fdfs/movie-service
EnvironmentFile=/etc/fdfs/movie-service.env
ExecStart=/opt/fdfs/movie-service/venv/bin/gunicorn --bind 0.0.0.0:8001 --workers 1 --threads 4 --access-logfile - app:app
Restart=on-failure
RestartSec=5
SyslogIdentifier=movie-service

[Install]
WantedBy=multi-user.target
```

> `--workers 1 --threads 4`: one process with 4 threads. Plenty for practice, and it keeps memory around 70 MB, so a t3.micro is enough.

## 9. Start and check

```shell
systemctl daemon-reload
systemctl enable --now fdfs-movie-service
systemctl status fdfs-movie-service --no-pager
```

```shell
curl -s localhost:8001/health
curl -s localhost:8001/ready
```

`/health` says the process is running. `/ready` also checks its dependencies and returns `503` if the database cannot be reached.

```shell
curl -s localhost:8001/api/movies
curl -s localhost:8001/api/movies
```

The first call says `"source": "database"`, the second `"source": "cache"`. That is Redis working.

## Logs

```shell
journalctl -u fdfs-movie-service -f
```
