# 📋 EVALUATION GUIDE — Inception (smikhail)

> This document is designed for the defense with your evaluator.
> Follow the steps in sequence and execute the exact commands for each section.

---

## 🍕 Credentials Quick Reference

| Service / Role | Username | Password | Where it is used |
|---|---|---|---|
| 👑 **WordPress Admin** | `smikhail_boss` | `pizza_margarita` | WP Admin Panel at `https://smikhail.42.fr/wp-admin` |
| 🛡️ **MariaDB Root** | `root` | `pizza_margarita` | Database CLI command: `docker exec -it mariadb mariadb -u root -ppizza_margarita` |
| 👤 **WordPress Regular User** | `smikhail_user` | `macarrones_con_tomate` | Adding comments at `https://smikhail.42.fr` |
| 🗄️ **MariaDB Database User** | `wp_user` | `macarrones_con_tomate` | Internal WordPress connection to MariaDB (port 3306) |

---

## ⚠️ STEP 0 — Initial Cleanup (Executed by the Evaluator)

The evaluator will execute this command in the terminal before starting:

```bash
docker stop $(docker ps -qa); docker rm $(docker ps -qa); docker rmi -f $(docker images -qa); docker volume rm $(docker volume ls -q); docker network rm $(docker network ls -q) 2>/dev/null
```

*Expected output:* Some errors may appear if there are no existing containers or volumes to remove. This is completely normal.

---

## 📁 STEP 1 — Clone the Repository

```bash
git clone <YOUR_GIT_REPO_URL> inception
cd inception
```

Verify directory structure:
```bash
ls -la
# Must show: Makefile  README.md  USER_DOC.md  DEV_DOC.md  srcs/  .gitignore

ls srcs/
# Must show: .env  docker-compose.yml  requirements/

ls srcs/requirements/
# Must show: nginx/  wordpress/  mariadb/
```

---

## 📄 STEP 2 — Verify README.md

```bash
head -1 README.md
```

✅ Exact first line (italicized with login):
```
*This project has been created as part of the 42 curriculum by smikhail*
```

Verify documentation files are present and not empty:
```bash
wc -l USER_DOC.md DEV_DOC.md
```

---

## 🔍 STEP 3 — Verify docker-compose.yml (Forbidden Terms)

```bash
cat srcs/docker-compose.yml
```

The evaluator verifies:
- ❌ No `network: host`
- ❌ No `links:`
- ✅ `networks:` is present (the `inception` network)

Check forbidden keywords directly (must return empty):
```bash
grep -n "network: host" srcs/docker-compose.yml
grep -n "links:" srcs/docker-compose.yml
```

---

## 🔍 STEP 4 — Verify Makefile and Scripts (No --link)

```bash
grep -r "\-\-link" Makefile srcs/
```

✅ Expected result: no output (empty)

---

## 🔍 STEP 5 — Verify Dockerfiles & Entrypoint Scripts

```bash
cat srcs/requirements/nginx/Dockerfile
cat srcs/requirements/wordpress/Dockerfile
cat srcs/requirements/mariadb/Dockerfile
```

The evaluator verifies:
- ✅ Each Dockerfile starts with `FROM debian:bookworm` (penultimate stable Debian release)
- ✅ No `tail -f`, `sleep infinity`, or unscripted `bash` in ENTRYPOINT
- ✅ No background processes (`&`) in ENTRYPOINT directives

Check scripts for forbidden infinite loops (must return empty):
```bash
grep -n "tail -f\|sleep infinity\|tail -f /dev/null" srcs/requirements/*/tools/*.sh
```

---

## 🚀 STEP 6 — Execute the Makefile

```bash
make
```

This will:
1. Create persistent host directories at `/home/smikhail/data/wordpress` and `/home/smikhail/data/mariadb`
2. Build all 3 Docker images from scratch
3. Start all containers in detached mode

Wait for the build to complete (approx. 2-4 minutes on first run).

---

## ✅ STEP 7 — Verify Running Containers

```bash
docker compose -f srcs/docker-compose.yml ps
```

Expected output:
```
NAME        IMAGE       STATUS    PORTS
mariadb     mariadb     running   3306/tcp
nginx       nginx       running   0.0.0.0:443->443/tcp
wordpress   wordpress   running   9000/tcp
```

---

## 🌐 STEP 8 — Verify NGINX (SSL/TLS Only)

```bash
# HTTP port 80 must NOT work (connection refused / timeout)
curl -k http://smikhail.42.fr

# HTTPS port 443 MUST work (returns WordPress HTML)
curl -k https://smikhail.42.fr | head -10
```

In the browser (inside VM): open **https://smikhail.42.fr**
- Self-signed certificate warning will appear -> click "Advanced" -> "Proceed anyway".
- The fully configured WordPress site must be displayed (NOT the installation setup page).

---

## 🔒 STEP 9 — Verify TLS 1.2 / TLS 1.3 Protocols

```bash
# Verify TLS 1.2 and TLS 1.3 work
openssl s_client -connect smikhail.42.fr:443 -tls1_2 2>/dev/null | grep "Protocol\|Cipher"
openssl s_client -connect smikhail.42.fr:443 -tls1_3 2>/dev/null | grep "Protocol\|Cipher"

# Verify insecure TLS 1.0 is blocked (must fail)
openssl s_client -connect smikhail.42.fr:443 -tls1 2>/dev/null | grep "handshake failure\|alert"
```

---

## 🐳 STEP 10 — Verify Docker Network

```bash
docker network ls
```

Expected result: the `inception` network is listed.

Inspect connected containers:
```bash
docker network inspect inception
```

All 3 containers (`nginx`, `wordpress`, `mariadb`) must be connected to this network.

**Explanation for the Evaluator:**
> The `inception` network is an isolated bridge network managed by Docker Compose. Containers communicate internally using service names as hostnames via Docker's built-in DNS server (e.g., WordPress connects to `mariadb:3306`, and NGINX forwards FastCGI requests to `wordpress:9000`). Externally, only port 443 of NGINX is published to the host system.

---

## 💾 STEP 11 — Verify WordPress Volume

```bash
docker volume ls
docker volume inspect wordpress_data
```

Check the JSON output contains:
```json
"Options": {
    "device": "/home/smikhail/data/wordpress",
    "o": "bind",
    "type": "none"
}
```

✅ `device` must point to `/home/smikhail/data/wordpress`.

---

## 💾 STEP 12 — Verify MariaDB Volume

```bash
docker volume inspect mariadb_data
```

Check the JSON output contains:
```json
"device": "/home/smikhail/data/mariadb"
```

✅ `device` must point to `/home/smikhail/data/mariadb`.

---

## 🌐 STEP 13 — Verify WordPress Web Interface

1. Open **https://smikhail.42.fr**
2. Confirm the WordPress site loads properly without showing any installation screen.
3. Post a comment using the regular user:
   - Username: `smikhail_user`
   - Password: `macarrones_con_tomate`

---

## 👤 STEP 14 — Verify WordPress Administrator Account

Open **https://smikhail.42.fr/wp-admin**

Admin credentials:
- Username: `smikhail_boss` (⚠️ must NOT contain "admin" or "Admin")
- Password: `pizza_margarita`

The evaluator verifies:
- ✅ Administrator dashboard is accessible
- ✅ Admin username does NOT contain "admin"
- ✅ Edit a page or create a test post, save changes, and verify the updated content on the live site

---

## 🗄️ STEP 15 — Verify MariaDB Database (Not Empty)

```bash
# Log in to MariaDB container as root
docker exec -it mariadb mariadb -u root -ppizza_margarita
```

Inside MariaDB shell:
```sql
SHOW DATABASES;
USE wordpress;
SHOW TABLES;
SELECT COUNT(*) FROM wp_posts;
exit
```

Expected output: `wordpress` database exists and contains standard WordPress tables (`wp_posts`, `wp_users`, `wp_comments`, etc.).

---

## 🔄 STEP 16 — Verify Data Persistence Across VM Reboot

```bash
# Stop containers
make down

# Reboot the virtual machine
sudo reboot
```

After the VM reboots:
```bash
cd ~/Desktop/INCEPTION
make
```

Check that:
1. `docker compose -f srcs/docker-compose.yml ps` shows all containers running.
2. Opening `https://smikhail.42.fr` immediately shows your configured site and the test post/comment from Step 14. No installation screen appears.

---

## ⚙️ STEP 17 — Service Configuration Modification (Port Change)

The evaluator will ask you to change a service configuration (e.g., changing NGINX HTTPS port from 443 to 8443):

```bash
# 1. Edit docker-compose.yml
nano srcs/docker-compose.yml
# Change port mapping under nginx:
# "443:443"  ->  "8443:443"

# 2. Rebuild and restart only the NGINX service
docker compose -f srcs/docker-compose.yml up -d --build nginx

# 3. Verify it is reachable on the new port
curl -k https://smikhail.42.fr:8443 | head -10
```

Revert back after verification:
```bash
nano srcs/docker-compose.yml   # Change back to "443:443"
docker compose -f srcs/docker-compose.yml up -d --build nginx
```

---

## 🎓 DEFENSE THEORY CHEAT SHEET

### 1. How do Docker and Docker Compose work?
> Docker uses Linux kernel namespaces and cgroups to package applications with their runtime dependencies into isolated containers. Docker Compose is an orchestration tool that manages multi-container applications defined in a YAML configuration, handling automatic network creation, volume mounting, and container lifecycle.

### 2. Difference between a Docker image used with and without Docker Compose?
> Without Docker Compose, images are built with `docker build` and run individually with `docker run`, requiring manual specification of every port, volume, and network flag. With Docker Compose, the declarative YAML file defines the entire topology, enabling single-command deployment (`docker compose up`) with automated dependency resolution.

### 3. Benefits of Docker compared to Virtual Machines?
> Containers share the host operating system's kernel, making them lightweight (MBs instead of GBs), fast to boot (seconds instead of minutes), and efficient in resource utilization. Virtual machines require a full guest operating system and hypervisor hardware emulation.

### 4. Why this specific directory structure?
> The structure follows the 42 subject requirements: all configuration files reside in `srcs/` separate from the root Makefile, and each service is encapsulated in its own subdirectory (`requirements/<service>/`) containing its dedicated Dockerfile and config files. This maintains separation of concerns and modularity.

### 5. What is a Docker Network?
> A Docker network provides isolated communication channels between containers on the same host using software bridge drivers. Containers on the network resolve each other by container/service name via Docker's embedded DNS server without exposing internal ports to the host system.
