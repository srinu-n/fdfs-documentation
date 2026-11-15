# 08 · frontend (Nginx)

**Server:** fdfs-web  **Port:** 80

Nginx does two jobs:

1. Serves the website files (`index.html`, `app.js`, `style.css`).
2. Works as a **reverse proxy**: each `/api/...` path is sent to the right service on the app server.

| Path | Goes to |
|---|---|
| `/api/movies` | movie-service on `<APP-SERVER-IP>:8001` |
| `/api/bookings` | booking-service on `<APP-SERVER-IP>:8002` |
| `/api/notifications` | notification-service on `<APP-SERVER-IP>:8004` |

The browser only ever talks to Nginx. It never knows the app server exists.

Run everything as root:

```shell
sudo -i
```

## 1. Install Nginx

```shell
apt update
apt install -y nginx unzip curl
```

## 2. Remove the default site

Ubuntu's default site also listens on port 80 and would clash with ours.

```shell
rm -f /etc/nginx/sites-enabled/default
```

## 3. Download the website

```shell
mkdir -p /var/www/fdfs
curl -o /tmp/frontend.zip https://fdfs-artifacts-30b67175.s3.amazonaws.com/frontend-v1.zip
cd /var/www/fdfs
unzip -o /tmp/frontend.zip
ls
```

## 4. Configure Nginx

```shell
vim /etc/nginx/conf.d/fdfs.conf
```

```nginx
server {
    listen 80 default_server;
    server_name _;

    root  /var/www/fdfs;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/movies {
        proxy_pass http://<APP-SERVER-IP>:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/bookings {
        proxy_pass http://<APP-SERVER-IP>:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/notifications {
        proxy_pass http://<APP-SERVER-IP>:8004;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location = /nginx-health {
        access_log off;
        return 200 "OK\n";
    }
}
```

> Replace all three `<APP-SERVER-IP>` with the private IP of fdfs-app.

## 5. Test the config and restart

Always test before restarting. A broken config stops Nginx from starting.

```shell
nginx -t
systemctl restart nginx
systemctl enable nginx
```

## 6. Check

From the web server:

```shell
curl -s localhost/nginx-health
curl -s localhost/api/movies
```

Then open `http://<WEB-SERVER-PUBLIC-IP>` in your browser. You should see the six shows, and the three service badges should be green.

## Logs

```shell
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```
