# 06 · booking-service

**Server:** fdfs-app  **Port:** 8002  **Language:** Python

booking-service takes a booking. It asks movie-service whether the show exists, asks payment-service to charge it, saves the booking in PostgreSQL, and publishes a `booking.created` message to RabbitMQ.

**Talks to:** movie-service (HTTP), payment-service (HTTP), PostgreSQL (`bookings` table) and RabbitMQ

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
mkdir -p /opt/fdfs/booking-service
```

## 4. Download the code

```shell
curl -o /tmp/booking-service.zip https://<ARTIFACTS-BUCKET>.s3.ap-south-1.amazonaws.com/booking-service-v1.zip
cd /opt/fdfs/booking-service
unzip -o /tmp/booking-service.zip
ls
```

## 5. Create a virtualenv and install packages

A virtualenv keeps this service's Python packages separate from the system and from the other services.

```shell
python3 -m venv /opt/fdfs/booking-service/venv
/opt/fdfs/booking-service/venv/bin/pip install -r /opt/fdfs/booking-service/requirements.txt
chown -R fdfs:fdfs /opt/fdfs/booking-service
```

## 6. Write the environment file

All settings come from environment variables. Nothing is hard-coded in the app.

```shell
mkdir -p /etc/fdfs
vim /etc/fdfs/booking-service.env
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
MOVIE_SERVICE_URL=http://localhost:8001
PAYMENT_SERVICE_URL=http://localhost:8003
```

```shell
chmod 600 /etc/fdfs/booking-service.env
```

> Replace `<DATA-SERVER-IP>` with the private IP of fdfs-data. `chmod 600` because the file holds passwords.

## 7. Load the table schema

`schema.sql` creates the `bookings` table.

```shell
PGPASSWORD=FdfsPass123 psql -h <DATA-SERVER-IP> -U fdfs -d fdfs -f /opt/fdfs/booking-service/db/schema.sql
```

## 8. Create the systemd service

```shell
vim /etc/systemd/system/fdfs-booking-service.service
```

```ini
[Unit]
Description=FDFS booking-service
After=network-online.target
Wants=network-online.target

[Service]
User=fdfs
Group=fdfs
WorkingDirectory=/opt/fdfs/booking-service
EnvironmentFile=/etc/fdfs/booking-service.env
ExecStart=/opt/fdfs/booking-service/venv/bin/gunicorn --bind 0.0.0.0:8002 --workers 1 --threads 4 --access-logfile - app:app
Restart=on-failure
RestartSec=5
SyslogIdentifier=booking-service

[Install]
WantedBy=multi-user.target
```

## 9. Start and check

```shell
systemctl daemon-reload
systemctl enable --now fdfs-booking-service
systemctl status fdfs-booking-service --no-pager
```

```shell
curl -s localhost:8002/health
curl -s localhost:8002/ready
```

`/health` says the process is running. `/ready` also checks its dependencies and returns `503` if the database cannot be reached.

```shell
curl -s -X POST localhost:8002/api/bookings \
  -H 'Content-Type: application/json' \
  -d '{"customer_name":"Ravi","movie_id":1,"seats":2}'
```

You should get the booking back with a `payment_id` and `"notification": "queued"`. movie-service and payment-service run on the same server, so their URLs use `localhost`. Install them first: this service needs both to make a booking.

## Logs

```shell
journalctl -u fdfs-booking-service -f
```
