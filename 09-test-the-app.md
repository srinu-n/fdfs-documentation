# 09 · Test the app

## Smoke test

Open `http://<WEB-SERVER-PUBLIC-IP>` and book a ticket.

1. The booking panel shows **Booking #1 confirmed** with a payment id.
2. About a second later a notification pops up: *Ticket confirmed: Ravi · 2 seat(s) · …*
3. The booking appears under **Recent bookings**.
4. All the service badges on the page are green.

The same checks from the command line, on the web server:

```shell
curl -s localhost/api/movies
curl -s -X POST localhost/api/bookings -H 'Content-Type: application/json' \
  -d '{"customer_name":"Ravi","movie_id":3,"seats":2}'
sleep 2
curl -s localhost/api/notifications
```

## Demos: watch the system handle failures

Each demo breaks one component on purpose. These are great to understand what each piece is for.

### 1. The cache

Reload the website. The "Now showing" panel says **served from PostgreSQL**, then **served from Redis cache** on the next reloads, for 30 seconds.

Stop Redis on the data server:

```shell
systemctl stop redis-server
```

The website still works and says **served from PostgreSQL** every time. The cache only makes things faster; it is not needed for correctness. Start it again with `systemctl start redis-server`.

### 2. The queue

Stop notification-service on the app server:

```shell
systemctl stop fdfs-notification-service
```

Make two or three bookings. Bookings still work, but no notifications appear and the notification-service badge turns red.

Open the RabbitMQ UI (`http://<DATA-SERVER-PUBLIC-IP>:15672`) → **Queues** → `booking.created`: the messages are waiting there.

Start it again:

```shell
systemctl start fdfs-notification-service
```

Within a few seconds all the missing notifications appear. Nothing was lost. That is why we put a queue between the two services.

### 3. A required dependency

Stop payment-service:

```shell
systemctl stop fdfs-payment-service
```

Try to book: **Booking failed: payment-service is not reachable, booking not made.** Unlike notifications, payment is required, so booking-service refuses the booking. Start it again with `systemctl start fdfs-payment-service`.

## Troubleshooting

| What you see | Most likely cause | Check |
|---|---|---|
| Website does not open | fdfs-web security group has no port 80, or Nginx is down | `systemctl status nginx` |
| Badge red, `502 Bad Gateway` | The service is down, wrong `<APP-SERVER-IP>` in Nginx, or fdfs-app security group is missing the port | `systemctl status fdfs-<service>`, `curl <APP-SERVER-IP>:<port>/health` from the web server |
| `/ready` says database `DOWN` | Wrong `DB_HOST`, port 5432 not allowed from fdfs-app, or `pg_hba.conf` missing the app server IP | `psql -h <DATA-SERVER-IP> -U fdfs -d fdfs` from the app server |
| Booking fails: movie-service or payment-service not reachable | That service is not running | `systemctl status fdfs-movie-service fdfs-payment-service` |
| Bookings work but no notifications | notification-service down, or cannot reach RabbitMQ | `curl localhost:8004/ready` (look at `rabbitmq_consumer`), `journalctl -u fdfs-notification-service` |
| Movies always "served from PostgreSQL" | Redis down or port 6379 not allowed | `curl localhost:8001/ready` (look at `redis`) |
| Service keeps restarting | A typo in the env file or a missing package | `journalctl -u fdfs-<service> -n 50` |
