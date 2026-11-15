# 02 · Redis

**Server:** fdfs-data  **Port:** 6379

movie-service caches the movie list in Redis for 30 seconds. The first request reads PostgreSQL; the next ones come from Redis.
If Redis is down, movie-service falls back to PostgreSQL, so the website keeps working, just slower.

Run everything as root:

```shell
sudo -i
```

## 1. Install Redis

```shell
apt update
apt install -y redis-server
```

## 2. Listen on the network

By default Redis only listens on `127.0.0.1` and runs in protected mode.

```shell
sed -i 's/^bind 127.0.0.1 -::1/bind 0.0.0.0/' /etc/redis/redis.conf
sed -i 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf
grep -E '^(bind|protected-mode)' /etc/redis/redis.conf
```

> Redis has no password here. The fdfs-data security group only allows port 6379 from fdfs-app, so nothing else can reach it.
> Adding a password is a good improvement for a later episode.

## 3. Restart and enable

```shell
systemctl restart redis-server
systemctl enable redis-server
ss -lntp | grep 6379
```

## 4. Check

```shell
redis-cli ping
```

You should see `PONG`.
