# Installing Immich with Podman, Pod, and Macvlan Network

## Introduction
This tutorial explains how to install **Immich** using **Podman**, leveraging **Pods** and **Macvlan network**.
The installation process requires **root access**. Adjust configurations as needed based on your environment.

> **Tested on:** Raspberry Pi 4 with OS Bookworm and Podman v4.3.1.

---

## 1. Creating a Macvlan Network
Run the following command to create a **Macvlan network** with the appropriate subnet, gateway, and interface:

```sh
sudo podman network create \
  --driver=macvlan \
  --subnet=192.168.18.0/24 \
  --gateway=192.168.18.1 \
  -o parent=eth0 \
  immich_network
```

> **Note:** Replace `eth0` and `ip` with your actual network interface.

---

## 2. Creating a Pod for Immich
Create a **Pod** with a **static IP** within the created network:

```sh
sudo podman pod create --name immich_pod \
  --add-host redis:127.0.0.1 \
  --add-host postgres:127.0.0.1 \
  --add-host machine_learning:127.0.0.1 \
  --network immich_network \
  --ip 192.168.18.177
```

---

## 3. Configuring the **.env File**
Create a `.env` file to store environment variables:

```sh
UPLOAD_LOCATION=/mnt/my_storage/immich/library
DB_DATA_LOCATION=/mnt/my_storage/immich/postgres

TZ=Asia/Jakarta

IMMICH_VERSION=release

DB_PASSWORD=postgres
DB_HOSTNAME=postgres
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

Ensure the `/mnt/my_storage/immich/` directory exists:

```sh
sudo mkdir -p /mnt/my_storage/immich/{library,postgres,model-cache}
sudo chown -R 1000:1000 /mnt/my_storage/immich/
```

---

## 4. Running Immich Containers
### 4.1 Redis
```sh
sudo podman run -d --pod immich_pod --name immich_redis \
  docker.io/redis:6.2-alpine
```

### 4.2 PostgreSQL with Vectors
```sh
sudo podman run -d --pod immich_pod --name immich_postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=immich \
  -e POSTGRES_INITDB_ARGS='--data-checksums' \
  -v /mnt/my_storage/immich/postgres:/var/lib/postgresql/data \
  docker.io/tensorchord/pgvecto-rs:pg14-v0.2.0 \
  postgres -c shared_preload_libraries=vectors.so \
           -c 'search_path="$$user", public, vectors' \
           -c logging_collector=on \
           -c max_wal_size=2GB \
           -c shared_buffers=512MB \
           -c wal_compression=on
```

### 4.3 Machine Learning Service
```sh
sudo podman run -d --pod immich_pod --name immich_machine_learning \
  -v /mnt/my_storage/immich/model-cache:/cache \
  --env-file .env \
  ghcr.io/immich-app/immich-machine-learning:release
```
You can change the `release`  to a specific version like "v1.130.3"

### 4.4 Immich Server
```sh
sudo podman run -d --pod immich_pod --name immich_server \
  -v /mnt/my_storage/immich/library:/usr/src/app/upload \
  -v /etc/localtime:/etc/localtime:ro \
  --env-file .env \
  ghcr.io/immich-app/immich-server:release
```
You can change the `release`  to a specific version like "v1.130.3"

---

## ~~5. Configuring **/etc/hosts** Inside the Server Container~~
~~Ensure proper service communication by adding the following entries:~~

**Ignore this**
```sh
#sudo podman exec immich_server sh -c "echo '127.0.0.1 redis' >> /etc/hosts"
#sudo podman exec immich_server sh -c "echo '127.0.0.1 postgres' >> /etc/hosts"
#sudo podman exec immich_server sh -c "echo '127.0.0.1 machine_learning' >> /etc/hosts"
```

---

## 6. Verifying Installation
### 6.1 Check Environment Variables
```sh
sudo podman exec immich_server env
```

### ~~6.2 Check **/etc/hosts**~~
**Ignore this**
```sh
sudo podman exec immich_server cat /etc/hosts
```

---

## 7. Accessing Immich via Browser
Open your browser and navigate to:

```
http://192.168.18.177:2283
```

---

## 8. Configuring Cloudflare Tunnel
To access Immich externally via **Cloudflare Tunnel**, run:

```sh
sudo podman run -d --name immich_tunnel-cloudflared --pod immich_pod \
  --restart always \
  docker.io/cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <your_token>
```

> **Replace `<your_token>` with your actual Cloudflare Tunnel token.**

---

## Completion 🎉
Immich is now running on **Podman**, using **Pods** and **Macvlan network**.
If you encounter issues, check running services with:

```sh
sudo podman ps --pod
```
```sh
sudo podman logs immich_server
```

---

### **References**
- [Immich Documentation](https://github.com/immich-app/immich)
- [Podman Documentation](https://podman.io/docs/)

