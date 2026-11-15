# 03 · RabbitMQ

**Server:** fdfs-data  **Ports:** 5672 (apps), 15672 (web UI)

booking-service publishes a `booking.created` message after every booking. notification-service reads those messages and turns them into notifications.
If notification-service is down, messages wait safely in the queue until it comes back.

Run everything as root:

```shell
sudo -i
```

## 1. Install RabbitMQ

```shell
apt update
apt install -y rabbitmq-server
systemctl enable --now rabbitmq-server
systemctl status rabbitmq-server --no-pager
```

## 2. Create the application user

The built-in `guest` user only works from localhost, so the services need their own user.

```shell
rabbitmqctl add_user fdfs FdfsPass123
rabbitmqctl set_permissions -p / fdfs ".*" ".*" ".*"
rabbitmqctl set_user_tags fdfs administrator
```

> The `administrator` tag lets you log in to the web UI with this user. In a real setup you would use a separate admin user.

## 3. Enable the web UI

```shell
rabbitmq-plugins enable rabbitmq_management
ss -lntp | grep -E '5672|15672'
```

Open `http://<DATA-SERVER-PUBLIC-IP>:15672` from your laptop and log in as `fdfs` / `FdfsPass123`.
(The security group allows 15672 only from your IP.)

The `booking.created` queue appears under **Queues** after the first booking.
