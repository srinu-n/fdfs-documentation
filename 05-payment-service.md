# 05 · payment-service

**Server:** fdfs-app  **Port:** 8003  **Language:** Java 21 (plain Java, built with Maven)

payment-service is a mock payment gateway. Every valid request is approved with a new payment id like `PAY-3F9A1C2B`. booking-service calls it before saving a booking.

It is internal only: no security group rule opens port 8003, because only booking-service (on the same server) calls it.

It is plain Java with no framework. It uses the small HTTP server that is built into the JDK, so the whole service is one Java file, the JAR is about 5 KB, and it runs in about 60 MB of memory.

**How Java is different from the Python services:** Python runs straight from the source code. Java has to be **built** first.
Maven reads `pom.xml`, compiles the code and packages it into one JAR file. We deploy only that JAR.

Run everything as root:

```shell
sudo -i
```

## 1. Install Java and Maven

Ubuntu 24.04's default JDK is Java 21.

```shell
apt update
apt install -y default-jdk-headless maven unzip curl
java -version
mvn -version
```

> The **JDK** (Java Development Kit) is needed to build. To only run a JAR, the smaller **JRE** is enough. This matters later for small Docker images.

## 2. Create the app user

```shell
id fdfs || useradd --system --no-create-home --shell /usr/sbin/nologin fdfs
```

## 3. Download the source code

We build in a temporary folder, not in the folder we run from.

```shell
mkdir -p /tmp/payment-service
curl -o /tmp/payment-service.zip https://fdfs-artifacts-30b67175.s3.amazonaws.com/payment-service-v1.zip
cd /tmp/payment-service
unzip -o /tmp/payment-service.zip
ls
```

You should see `pom.xml` and `src/`.

## 4. Build the JAR with Maven

```shell
cd /tmp/payment-service
mvn clean package -DskipTests
ls -lh target/payment-service.jar
```

> The first build downloads Maven's plugins into `~/.m2` and takes a minute. The next builds take a few seconds.
> `clean` removes old build output, `package` compiles the code and creates the JAR in `target/`.

## 5. Copy only the JAR

```shell
mkdir -p /opt/fdfs/payment-service
cp /tmp/payment-service/target/payment-service.jar /opt/fdfs/payment-service/app.jar
chown -R fdfs:fdfs /opt/fdfs/payment-service
```

## 6. Write the environment file

```shell
mkdir -p /etc/fdfs
echo "SERVER_PORT=8003" > /etc/fdfs/payment-service.env
chmod 600 /etc/fdfs/payment-service.env
```

## 7. Create the systemd service

```shell
vim /etc/systemd/system/fdfs-payment-service.service
```

```ini
[Unit]
Description=FDFS payment-service
After=network-online.target
Wants=network-online.target

[Service]
User=fdfs
Group=fdfs
WorkingDirectory=/opt/fdfs/payment-service
EnvironmentFile=/etc/fdfs/payment-service.env
ExecStart=/usr/bin/java -Xms32m -Xmx64m -jar /opt/fdfs/payment-service/app.jar
Restart=on-failure
RestartSec=5
SyslogIdentifier=payment-service

[Install]
WantedBy=multi-user.target
```

> `-Xms32m -Xmx64m` sets the JVM heap: start with 32 MB, never use more than 64 MB.
> Without it, Java sizes its heap from the server's RAM and takes more memory than this small service needs.

## 8. Start and check

```shell
systemctl daemon-reload
systemctl enable --now fdfs-payment-service
systemctl status fdfs-payment-service --no-pager
```

> **Installing a new version later?** `enable --now` does not restart a service that is already running.
> After replacing the code, run `systemctl restart fdfs-payment-service`, otherwise the old version keeps running.

```shell
curl -s localhost:8003/health
curl -s -X POST localhost:8003/api/payments \
  -H 'Content-Type: application/json' \
  -d '{"amount": 300}'
```

You should see `"status":"PAID"` and a `payment_id`.

## Logs

```shell
journalctl -u fdfs-payment-service -f
```
