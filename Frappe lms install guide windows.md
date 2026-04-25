# Frappe Framework + LMS + Payments — Complete Installation Guide
> Windows 10/11 Local Development Setup (via WSL2)

---

## Important Note

> Frappe does **not** run natively on Windows. The official and recommended approach is to use **WSL2 (Windows Subsystem for Linux)** which gives you a full Ubuntu environment inside Windows.

---

## Step 1 — Enable WSL2

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

This installs WSL2 with Ubuntu by default. Restart your computer when prompted.

After restart, Ubuntu will launch and ask you to create a Linux username and password:
```
Enter new UNIX username: frappe
Enter new UNIX password: yourpassword
```

> ⚠️ This Linux password is separate from your Windows password. Remember it — you'll need it for `sudo` commands.

### Verify WSL2 is installed correctly
```powershell
wsl --list --verbose
# Should show Ubuntu running with VERSION 2
```

---

## Step 2 — Open Ubuntu Terminal

You can open your Ubuntu WSL terminal in any of these ways:
- Search **Ubuntu** in the Start Menu
- Open **Windows Terminal** and select Ubuntu from the dropdown
- Run `wsl` in PowerShell

All the following commands are run **inside the Ubuntu WSL terminal**.

---

## Step 3 — Update Ubuntu

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

---

## Step 4 — Install System Dependencies

```bash
sudo apt-get install -y \
  python3-dev python3-pip python3-venv \
  git curl wget \
  libmariadb-dev pkg-config \
  libffi-dev libssl-dev \
  redis-server \
  nodejs npm \
  xvfb libfontconfig wkhtmltopdf

npm install -g yarn
```

---

## Step 5 — MariaDB Setup

> ⚠️ Frappe requires **MariaDB 10.6+**. Do NOT use MySQL — it is not supported.

### Install MariaDB
```bash
sudo apt-get install -y mariadb-server mariadb-client
```

### Start MariaDB
```bash
sudo service mariadb start
```

> Note: WSL2 does not use `systemctl`. Use `sudo service` instead.

### Set Root Password
```bash
sudo mysql -u root
```
Inside the MariaDB shell:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'yourpassword';
FLUSH PRIVILEGES;
EXIT;
```

### Configure Charset for Frappe
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
sudo service mariadb restart
```

### Verify MariaDB
```bash
mysql --version
# Should show: MariaDB 10.6.x — NOT MySQL
mysql -u root -pyourpassword -e "SELECT VERSION();"
```

---

## Step 6 — Install Frappe Bench CLI

```bash
pip3 install frappe-bench --break-system-packages

# Add to PATH if bench is not found
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify
bench --version
```

---

## Step 7 — Initialize Bench

```bash
bench init frappe-bench --frappe-branch version-15 --python python3
cd frappe-bench
```

> `version-15` is the stable LTS branch supporting Python 3.11/3.12.

---

## Step 8 — Get Apps

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

## Step 9 — Create a Site

```bash
bench new-site lms.localhost \
  --db-root-password yourpassword \
  --admin-password yourpassword
```

---

## Step 10 — Install Apps on the Site

```bash
# Install payments first (LMS depends on it)
bench --site lms.localhost install-app payments

# Install LMS
bench --site lms.localhost install-app lms
```

---

## Step 11 — Configure & Start

### 1. Set Default Site
```bash
bench use lms.localhost
```

### 2. Add to hosts file
```bash
# Inside WSL
echo "127.0.0.1 lms.localhost" | sudo tee -a /etc/hosts
```

Also add it to Windows hosts file — open **Notepad as Administrator** and edit:
```
C:\Windows\System32\drivers\etc\hosts
```
Add this line:
```
127.0.0.1 lms.localhost
```

### 3. Start Redis
```bash
sudo service redis-server start
```

### 4. Start Bench
```bash
bench start
```

---

## Access Your Site

Open your **Windows browser** and go to:

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
| MariaDB not running | `sudo service mariadb start` |
| Redis not running | `sudo service redis-server start` |
| Port 8000 in use | `lsof -i :8000` then `kill -9 <PID>` |
| WSL2 not version 2 | `wsl --set-version Ubuntu 2` |
| Can't reach site in browser | Make sure hosts file is updated in **both** WSL and Windows |

---

## WSL2 Tips

- Access your Linux files from Windows at: `\\wsl$\Ubuntu\home\<username>`
- Access your Windows files from WSL at: `/mnt/c/Users/<username>`
- To restart WSL: run `wsl --shutdown` in PowerShell then reopen Ubuntu
- Always run `sudo service mariadb start` and `sudo service redis-server start` after a WSL restart — services don't auto-start in WSL2

---

## Full Command Summary

```bash
# Run inside Ubuntu WSL terminal

# 1. System dependencies
sudo apt-get update
sudo apt-get install -y python3-dev python3-pip python3-venv git curl \
  libmariadb-dev pkg-config libffi-dev libssl-dev \
  redis-server nodejs npm mariadb-server mariadb-client
npm install -g yarn
pip3 install frappe-bench --break-system-packages

# 2. MariaDB
sudo service mariadb start
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
sudo service redis-server start
bench start
```

---

*Guide based on Frappe version-15 + LMS v2.52 on Windows 10/11 via WSL2 (Ubuntu 22.04)*