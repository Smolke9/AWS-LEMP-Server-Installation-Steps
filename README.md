# LEMP on AWS Linux (Amazon Linux 2 & 2023) — Step‑by‑Step Guide

This README walks you through installing and configuring **LEMP** (Linux, Nginx, MariaDB, PHP) on an **Amazon EC2** instance running **Amazon Linux 2** or **Amazon Linux 2023**.

> ✅ Works on a fresh EC2 instance. Commands require `sudo` privileges.
> ✅ Uses `systemctl` (recommended) instead of the older `service` command.
> ✅ Includes Nginx + PHP‑FPM config, MariaDB secure setup, and a test PHP page.

---

## 0) Prerequisites

1. **Launch an EC2 instance**
   - AMI: **Amazon Linux 2** (recommended) or **Amazon Linux 2023**  
   - Instance type: `t2.micro`/`t3.micro` (for testing)
   - Key pair: your `.pem` key
   - Storage: ≥ 8 GB (recommended)

2. **Security Group (very important)**
   - Inbound rules:
     - **SSH**: TCP **22** from your IP
     - **HTTP**: TCP **80** from `0.0.0.0/0`
     - **HTTPS**: TCP **443** from `0.0.0.0/0` (for TLS, optional now)
     - *(Optional)* **MySQL/Aurora**: TCP **3306** — **do NOT** open to the world. Restrict to trusted sources only.

3. **Connect via SSH**
   ```bash
   ssh -i /path/to/your-key.pem ec2-user@<EC2_PUBLIC_IP>
   ```

---

## 1) Update the OS

### Amazon Linux 2
```bash
sudo yum update -y
```

### Amazon Linux 2023
```bash
sudo dnf update -y
```

---

## 2) Install Nginx, MariaDB, PHP‑FPM

### Amazon Linux 2
```bash
# Nginx + MariaDB + PHP-FPM + common PHP extensions
sudo yum install -y nginx mariadb-server php php-fpm php-mysqlnd php-cli php-json php-zip php-gd php-mbstring php-curl php-xml
```

### Amazon Linux 2023
```bash
sudo dnf install -y nginx mariadb105-server php php-fpm php-mysqlnd php-cli php-json php-zip php-gd php-mbstring php-curl php-xml
```

> If `mariadb105-server` is not available on your AL2023 build, use `mariadb-server` instead.

---

## 3) Start and enable services

```bash
# Stop Apache if it’s installed (to avoid port conflicts)
sudo systemctl stop httpd 2>/dev/null || true
sudo systemctl disable httpd 2>/dev/null || true

# Nginx
sudo systemctl enable nginx
sudo systemctl start nginx

# MariaDB
sudo systemctl enable mariadb
sudo systemctl start mariadb

# PHP‑FPM
sudo systemctl enable php-fpm
sudo systemctl start php-fpm
```

Check status:
```bash
systemctl is-active nginx mariadb php-fpm
```

---

## 4) Secure MariaDB

Run the built‑in hardening tool:
```bash
sudo mysql_secure_installation
```
Recommended answers:
- Set root password: **Y** (and choose a strong one)
- Remove anonymous users: **Y**
- Disallow remote root login: **Y**
- Remove test database: **Y**
- Reload privilege tables: **Y**

Create an app database and user (example):
```bash
sudo mysql -u root -p -e "\
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci; \
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'Strong_Passw0rd!'; \
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost'; \
FLUSH PRIVILEGES;"
```

---

## 5) Set up document root

Create a simple web root (you can change this path):
```bash
sudo mkdir -p /var/www/app/public
echo '<?php phpinfo();' | sudo tee /var/www/app/public/info.php > /dev/null
```

Ownership & permissions:
```bash
# Give Nginx read access; keep ec2-user as editor
sudo chown -R ec2-user:nginx /var/www/app
sudo find /var/www/app -type d -exec chmod 750 {} \;
sudo find /var/www/app -type f -exec chmod 640 {} \;
```

*(Optional)* Allow your SSH user to edit without sudo:
```bash
sudo usermod -aG nginx ec2-user
# Log out and back in to apply group change
```

---

## 6) Configure PHP‑FPM (pool)

Open `/etc/php-fpm.d/www.conf`:
```bash
sudo nano /etc/php-fpm.d/www.conf
```
Recommended changes (ensure these values exist and are uncommented):
```
user = nginx
group = nginx

; For socket permissions
listen = /run/php-fpm/www.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

Restart PHP‑FPM after changes:
```bash
sudo systemctl restart php-fpm
```

---

## 7) Configure Nginx (server block)

Create a new site config, e.g. `/etc/nginx/conf.d/app.conf`:
```bash
sudo nano /etc/nginx/conf.d/app.conf
```

Paste this minimal PHP‑enabled server block (replace `SERVER_NAME` if you have a domain):
```
server {
    listen 80;
    server_name _;  # or yourdomain.com
    root /var/www/app/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        include fastcgi_params;
    }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|webp)$ {
        expires 7d;
        access_log off;
        add_header Cache-Control "public";
    }

    # Security/Hardening examples
    location ~ /\.ht {
        deny all;
    }
}
```

Test and reload Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8) Verify PHP is working

Open your browser to:
```
http://<EC2_PUBLIC_IP>/info.php
```
You should see the PHP info page. **Remember to delete it later**:
```bash
sudo rm -f /var/www/app/public/info.php
```

---

## 9) Optional: Enable HTTPS (Let’s Encrypt)

> For production, always use HTTPS. Below are common approaches; choose one depending on your AMI tooling availability.

**Option A — Certbot via Snap (often easiest):**
```bash
# Install snapd (AL2/AL2023 may need EPEL first)
sudo yum -y install snapd || sudo dnf -y install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Obtain and install certs for Nginx (replace domain and email)
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com --agree-tos -m you@example.com --redirect
```

**Option B — Certbot from repos:**  
Install the `certbot` + `python3-certbot-nginx` packages if available for your AMI and run:
```bash
sudo certbot --nginx -d yourdomain.com --agree-tos -m you@example.com --redirect
```

Auto‑renew is usually set by a systemd timer. Test renewal:
```bash
sudo certbot renew --dry-run
```

---

## 10) Useful service commands

```bash
# Start/Stop/Restart
sudo systemctl start nginx
sudo systemctl restart nginx
sudo systemctl status nginx

sudo systemctl start mariadb
sudo systemctl restart mariadb
sudo systemctl status mariadb

sudo systemctl start php-fpm
sudo systemctl restart php-fpm
sudo systemctl status php-fpm

# View Nginx logs
sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

---

## 11) Troubleshooting

- **Port 80 not reachable**
  - Check AWS **Security Group** inbound rules include TCP **80** from `0.0.0.0/0`.
  - Ensure Nginx is running: `systemctl status nginx`.
- **`nginx: [emerg] bind() to 0.0.0.0:80 failed`**
  - Another service (e.g., Apache) is using port 80. Stop it:
    ```bash
    sudo systemctl stop httpd
    sudo systemctl disable httpd
    sudo systemctl restart nginx
    ```
- **`502 Bad Gateway`**
  - PHP‑FPM not running or socket mismatch.
  - Ensure `listen = /run/php-fpm/www.sock` in PHP‑FPM and the same path in Nginx `fastcgi_pass`.
  - Restart services: `systemctl restart php-fpm nginx`.
- **PHP file downloads instead of executing**
  - Missing or incorrect `location ~ \.php$` block.
  - Ensure `fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;` is present.
- **Permission denied writing uploads**
  - Ensure directory ownership/permissions allow Nginx/PHP‑FPM to write where needed.
  - Example:
    ```bash
    sudo chown -R ec2-user:nginx /var/www/app
    sudo chmod -R 775 /var/www/app/storage   # adjust to your writable path
    ```

---

## 12) One‑shot quick script (Amazon Linux 2)

> Use this if you just want to set up a quick test instance.

```bash
#!/usr/bin/env bash
set -e

# Update
sudo yum update -y

# Install
sudo yum install -y nginx mariadb-server php php-fpm php-mysqlnd php-cli php-json php-zip php-gd php-mbstring php-curl php-xml

# Stop Apache if present
sudo systemctl stop httpd 2>/dev/null || true
sudo systemctl disable httpd 2>/dev/null || true

# Enable services
sudo systemctl enable nginx mariadb php-fpm
sudo systemctl start nginx mariadb php-fpm

# Secure MariaDB (non-interactive example – adjust password!)
MYSQL_ROOT_PW='StrongRoot_Passw0rd!'
sudo mysql -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '${MYSQL_ROOT_PW}'; FLUSH PRIVILEGES;" || true

# App root
sudo mkdir -p /var/www/app/public
echo '<?php phpinfo();' | sudo tee /var/www/app/public/info.php >/dev/null
sudo chown -R ec2-user:nginx /var/www/app
sudo find /var/www/app -type d -exec chmod 750 {} \;
sudo find /var/www/app -type f -exec chmod 640 {} \;

# PHP‑FPM pool tweaks
sudo sed -i 's/^user = .*/user = nginx/' /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = .*/group = nginx/' /etc/php-fpm.d/www.conf
sudo sed -i 's|^listen = .*|listen = /run/php-fpm/www.sock|' /etc/php-fpm.d/www.conf
if ! grep -q "^listen.owner" /etc/php-fpm.d/www.conf; then
  echo -e "listen.owner = nginx\nlisten.group = nginx\nlisten.mode = 0660" | sudo tee -a /etc/php-fpm.d/www.conf
fi
sudo systemctl restart php-fpm

# Nginx server block
sudo tee /etc/nginx/conf.d/app.conf >/dev/null <<'NGINX'
server {
    listen 80;
    server_name _;
    root /var/www/app/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include /etc/nginx/fastcgi_params;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
        include fastcgi_params;
    }

    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|webp)$ {
        expires 7d;
        access_log off;
        add_header Cache-Control "public";
    }

    location ~ /\.ht {
        deny all;
    }
}
NGINX

sudo nginx -t
sudo systemctl reload nginx

echo "LEMP ready. Visit http://<EC2_PUBLIC_IP>/info.php"
```

---

## 13) Clean up the test page

Once you confirm PHP works, remove the info page:
```bash
sudo rm -f /var/www/app/public/info.php
```

---

### That’s it!
You now have **Nginx + MariaDB + PHP‑FPM** running on AWS Linux. Deploy your app into `/var/www/app/public` or adjust paths to match your project.
