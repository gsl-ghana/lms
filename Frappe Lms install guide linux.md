# Frappe Framework + LMS + Payments — Complete Installation Guide
> Linux (Ubuntu 22.04 / 24.04) Local Development Setup

---

## Prerequisites

### 1. Update System
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 2. Install System Dependencies
```bash
sudo apt-get install -y \
  python3-dev python3-pip python3-venv \
  git curl wget \
  libmariadb-dev pkg-config \
  libffi-dev libssl-dev \
  redis-server \
  nodejs npm \
  xvfb libfontconfig wkhtmltopdf
```

### 3. Install Yarn
```bash
npm install -g yarn
```

---

## MariaDB Setup

> ⚠️ Frappe requires **MariaDB 10.6+**. Do NOT use MySQL — it is not supported.

### 1. Install MariaDB 10.6
```bash
sudo apt-get install -y mariadb-server mariadb-client
```

### 2. Start MariaDB
```bash
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

### 3. Secure MariaDB & Set Root Password
```bash
sudo mysql_secure_installation
# → Set root password (e.g. yourpassword)
# → Answer Y to all prompts
```

Or manually:
```bash
sudo mysql -u root
```
Inside the MariaDB shell:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'yourpassword';
FLUSH PRIVILEGES;
EXIT;
```

### 4. Configure Charset for Frappe
```bash
sudo nano /etc/mysql/conf.d/frappe.cnf
```
Add the following:
```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```
```bash
sudo systemctl restart mariadb
```

### 5. Verify MariaDB is Running
```bash
mysql --version
# Should show: MariaDB 10.6.x — NOT MySQL
mysql -u root -pyourpassword -e "SELECT VERSION();"
```

---

## Install Frappe Bench CLI

```bash
pip3 install frappe-bench --break-system-packages

# Verify
bench --version
```

> If `bench` is not found after install:
> ```bash
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> ```

---

## Initialize Bench

```bash
# Create your bench workspace
bench init frappe-bench --frappe-branch version-15 --python python3

# Go into the bench directory
cd frappe-bench
```

> `version-15` is the stable LTS branch supporting Python 3.11/3.12.

---

## Get Apps

### 1. Get Payments App (required by LMS)
```bash
bench get-app payments --branch version-15
```

### 2. Get LMS App
```bash
bench get-app lms https://github.com/frappe/lms --branch main
```

Your `apps/` folder should now look like:
```
frappe-bench/
└── apps/
    ├── frappe/       ✅ Core framework
    ├── payments/     ✅ Payments dependency
    └── lms/          ✅ LMS app
```

---

## Create a Site

```bash
bench new-site lms.localhost \
  --db-root-password yourpassword \
  --admin-password yourpassword
```

What this does:
- Creates a MariaDB database for the site
- Creates a dedicated DB user
- Installs all core Frappe tables

---

## Install Apps on the Site

```bash
# Install payments first (LMS depends on it)
bench --site lms.localhost install-app payments

# Install LMS
bench --site lms.localhost install-app lms
```

---

## Configure & Start

### 1. Set Default Site
```bash
bench use lms.localhost
```

### 2. Add to /etc/hosts
```bash
echo "127.0.0.1 lms.localhost" | sudo tee -a /etc/hosts
```

### 3. Start Redis
```bash
sudo systemctl start redis-server
```

### 4. Start Bench
```bash
bench start
```

This starts:
- `web` → Frappe web server on port 8000
- `worker` → Background job workers
- `redis` → Cache and queue
- `scheduler` → Cron-like task runner

---

## Access Your Site

Open your browser and go to:

```
http://lms.localhost:8000
```

| Field    | Value           |
|----------|-----------------|
| Username | `Administrator` |
| Password | `yourpassword`  |

---

## User Roles

Once logged in, go to **Settings → User** to create users and assign roles:

| Role            | Access                          |
|-----------------|---------------------------------|
| System Manager  | Full admin access               |
| LMS Instructor  | Create and manage courses       |
| LMS Student     | Enroll and take courses         |

---

## Quick Troubleshooting

| Error | Fix |
|-------|-----|
| `bench` not found | `export PATH="$HOME/.local/bin:$PATH"` |
| `Access denied for user 'root'` | Use `sudo mysql -u root` for first login |
| `No module named 'payments'` | Run `bench get-app payments --branch version-15` first |
| `Database already exists` | `mysql -u root -p -e "DROP DATABASE \`lms_localhost\`;"` |
| MariaDB version warning | Make sure `mariadb-server` is 10.6+ not MySQL |
| Port 8000 in use | `lsof -i :8000` then `kill -9 <PID>` |
| Redis not running | `sudo systemctl start redis-server` |

---

## Full Command Summary

```bash
# 1. System dependencies
sudo apt-get update
sudo apt-get install -y python3-dev python3-pip python3-venv git curl \
  libmariadb-dev pkg-config libffi-dev libssl-dev redis-server nodejs npm
npm install -g yarn
pip3 install frappe-bench --break-system-packages

# 2. MariaDB
sudo apt-get install -y mariadb-server mariadb-client
sudo systemctl start mariadb
sudo mysql -u root
# → SET PASSWORD, FLUSH PRIVILEGES, EXIT

# 3. Bench
bench init frappe-bench --frappe-branch version-15 --python python3
cd frappe-bench

# 4. Apps
bench get-app payments --branch version-15
bench get-app lms https://github.com/frappe/lms --branch main

# 5. Site
bench new-site lms.localhost --db-root-password yourpassword --admin-password yourpassword

# 6. Install apps on site
bench --site lms.localhost install-app payments
bench --site lms.localhost install-app lms

# 7. Start
bench use lms.localhost
echo "127.0.0.1 lms.localhost" | sudo tee -a /etc/hosts
bench start
```

---

*Guide based on Frappe version-15 + LMS v2.52 on Ubuntu 22.04 / 24.04*