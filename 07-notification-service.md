# 07 · notification-service

**Server:** fdfs-app  **Port:** 8004  **Language:** Python

notification-service does two jobs in one process: a background consumer reads `booking.created` messages from RabbitMQ and saves a notification in PostgreSQL, and a small API serves those notifications to the website.

**Talks to:** RabbitMQ and PostgreSQL (`notifications` table)

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
mkdir -p /opt/fdfs/notification-service
```

## 4. Download the code

```shell
curl -o /tmp/notification-service.zip https://fdfs-artifacts-30b67175.s3.amazonaws.com/notification-service-v1.zip
cd /opt/fdfs/notification-service
unzip -o /tmp/notification-service.zip
ls
```

## 5. Create a virtualenv and install packages

A virtualenv keeps this service's Python packages separate from the system and from the other services.

```shell
python3 -m venv /opt/fdfs/notification-service/venv
/opt/fdfs/notification-service/venv/bin/pip install -r /opt/fdfs/notification-service/requirements.txt
chown -R fdfs:fdfs /opt/fdfs/notification-service
```

## 6. Write the environment file

All settings come from environment variables. Nothing is hard-coded in the app.

```shell
mkdir -p /etc/fdfs
vim /etc/fdfs/notification-service.env
```

```ini
DB_HOST=<DATA-SERVER-IP>
DB_PORT=5432
DB_NAME=fdfs
DB_USER=fdfs
DB_PASSWORD=FdfsPass123
RABBITMQ_HOST=<DATA-SERVER-IP>
RABBITMQ_PORT=5672
RABBITMQ_USER=fdfs
RABBITMQ_PASSWORD=FdfsPass123
```

```shell
chmod 600 /etc/fdfs/notification-service.env
```

> Replace `<DATA-SERVER-IP>` with the private IP of fdfs-data. `chmod 600` because the file holds passwords.

## 7. Load the table schema

`schema.sql` creates the `notifications` table.

```shell
PGPASSWORD=FdfsPass123 psql -h <DATA-SERVER-IP> -U fdfs -d fdfs -f /opt/fdfs/notification-service/db/schema.sql
```

## 8. Create the systemd service

```shell
vim /etc/systemd/system/fdfs-notification-service.service
```

```ini
[Unit]
Description=FDFS notification-service
After=network-online.target
Wants=network-online.target

[Service]
User=fdfs
Group=fdfs
WorkingDirectory=/opt/fdfs/notification-service
EnvironmentFile=/etc/fdfs/notification-service.env
ExecStart=/opt/fdfs/notification-service/venv/bin/gunicorn --bind 0.0.0.0:8004 --workers 1 --threads 4 --access-logfile - app:app
Restart=on-failure
RestartSec=5
SyslogIdentifier=notification-service

[Install]
WantedBy=multi-user.target
```

> Keep this service at **one** gunicorn worker (`--workers 1`). Each worker would start its own RabbitMQ consumer, and we want exactly one. The 4 threads handle the web requests.

## 9. Start and check

```shell
systemctl daemon-reload
systemctl enable --now fdfs-notification-service
systemctl status fdfs-notification-service --no-pager
```

> **Installing a new version later?** `enable --now` does not restart a service that is already running.
> After replacing the code, run `systemctl restart fdfs-notification-service`, otherwise the old version keeps running.

```shell
curl -s localhost:8004/health
curl -s localhost:8004/ready
```

`/health` says the process is running. `/ready` also checks its dependencies and returns `503` if the database cannot be reached.

```shell
curl -s localhost:8004/api/notifications
```

If you made a booking in the previous step, its notification is already here. It was waiting in the RabbitMQ queue until this service started.

## Logs

```shell
journalctl -u fdfs-notification-service -f
```
