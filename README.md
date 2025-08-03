# Installing And Configuring Of Wordpress, NGINX, MARIADB, PHPmyadmin, PHP, CloudFlared Tunnel In Linux.

---

## **1. Update Your System**
```bash
sudo apt update
sudo apt upgrade
```

---

## **2. Install Nginx**
```bash
sudo apt install nginx
sudo systemctl status nginx
```

---

## **3. Install and Configure MariaDB**
```bash
sudo apt install mariadb-server
sudo systemctl start mariadb
sudo systemctl status mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation   
```

### Access MariaDB and run the following:  **[START FROM HERE TO HOST ANOTHER WORDPRESS WEBSITE]**
```bash
sudo mysql -u root -p
```

### Inside the MariaDB shell:
```sql
CREATE DATABASE db_<WEBSITE NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'admin@<WEBSITE NAME>'@'localhost' IDENTIFIED BY '<WEBSITE NAME>@<PASSWORD>';
GRANT ALL PRIVILEGES ON db_<WEBSITE NAME>.* TO 'admin@<WEBSITE NAME>'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## **4. Install PHP and phpMyAdmin**          **[SKIP  STEP 4 IF HOSTING ANOTHER WORDPRESS WEBSITE]**
```bash
sudo apt install php-fpm php-mysql
sudo apt install phpmyadmin
```

- **Skip** the web server selection (Apache/Lighttpd).
- Select **No** when asked to configure the database with `dbconfig-common`.

---

## **5. Download and Configure WordPress**
```bash
cd /var/www/
sudo mkdir <WEBSITE NAME>
cd <WEBSITE NAME>
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xvzf latest.tar.gz
sudo mv wordpress public_html   
sudo chown -R www-data:www-data public_html/
sudo chmod -R 755 public_html/
sudo rm -r latest.tar.gz
cd public_html
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

- Change the **Database Name**, **User Name**, and **Password**.
- Add salts from: [Wordpress Salts Download](https://api.wordpress.org/secret-key/1.1/salt/)

---

## **6. Configure Nginx for WordPress**
```bash
sudo nano /etc/nginx/sites-available/<WEBSITE NAME>
```

### Paste the following config:
```nginx
upstream <WEBSITE_NAME>-php-handler {
    server unix:/var/run/php/php8.2-fpm.sock;
}

server {
    listen 443 ssl;
    server_name <WEBSITE_DOMAIN>;

    ssl_certificate /etc/ssl/certs/<WEBSITE_DOMAIN>/origin.crt;
    ssl_certificate_key /etc/ssl/private/<WEBSITE_DOMAIN>/origin.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 5120M;

    root /var/www/<WEBSITE_NAME>/public_html; 
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass <WEBSITE_NAME>-php-handler;
    }

    location /phpmyadmin {
        root /usr/share/;
        index index.php index.html index.htm;

        location ~ ^/phpmyadmin/(.+\.php)$ {
            root /usr/share/;
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        }

        location ~* ^/phpmyadmin/(.+\.(jpg|jpeg|gif|css|png|js|ico|html|xml|txt))$ {
            root /usr/share/;
        }
    }
}

server {
    listen 80;
    server_name <WEBSITE_DOMAIN>;
    return 301 https://$host$request_uri;
}
```


## **7. Obtain and Install Cloudflare Origin Certificate (SSL) [RECOMMENDED & Secure]**

Use **one certificate** for your main domain and all first-level subdomains.

---

### 🔐 On Cloudflare Dashboard:

1. Go to **SSL/TLS → Origin Server** for your root domain.
2. Click **Create Certificate**.
3. Keep Everything Default
4. Click **Create**.
5. Copy the **Origin Certificate** and **Private Key**.

---

### 🖥️ On Your Server:

Create directories and install certificate files:

```bash
sudo mkdir -p /etc/ssl/certs/<WEBSITE_DOMAIN>
sudo mkdir -p /etc/ssl/private/<WEBSITE_DOMAIN>

sudo nano /etc/ssl/certs/<WEBSITE_DOMAIN>/origin.crt
```
Paste the Origin Certificate here
```bash
sudo nano /etc/ssl/private/<WEBSITE_DOMAIN>/origin.key
```
Paste the Private Key here
```bash

sudo chmod 600 /etc/ssl/private/<WEBSITE_DOMAIN>/origin.key
sudo chown root:root /etc/ssl/private/<WEBSITE_DOMAIN>/origin.key
```

### Then run:
```bash
sudo ln -s /etc/nginx/sites-available/<WEBSITE NAME> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx     
```

✅ Your SSL certificate is now installed and ready to be referenced in your NGINX site configuration!

---
## **8. Configure Tunnel Public Hostname (HTTPS Access via Cloudflare Tunnel)**

Secure your local WordPress site over HTTPS using a Cloudflare Tunnel.

---

### ⚙️ Cloudflare Zero Trust Dashboard Setup

1. Go to **Zero Trust Dashboard → Access → Tunnels**  
2. Select your existing tunnel  
3. Click **Add/Edit Public Hostname**

---

### 🔧 Fill in the Public Hostname Details:

- **Service Type:** `HTTPS`   
- **URL:**  
  ```
  localhost:443
  ```
  
---

### 🛡️ TLS Settings

- Click **Additional Application Settings → TLS**
- Enable:
  - ✅ **No TLS Verify** (prevents self-signed cert issues)

---

### 💾 Save and Deploy

Once saved, Cloudflare will route encrypted HTTPS traffic through your tunnel to the local server securely. ✅

---

## **9. Increase Max File Upload Limit to 10GB**

To allow large uploads (up to 10 GB), modify PHP configuration.

### Step 1: Edit `php.ini` for PHP-FPM
```bash
sudo nano /etc/php/8.2/fpm/php.ini
```

### In the file, search using `Ctrl + W` in nano:
- Search for: `upload_max_filesize` → change to `5120M` # ✅ 5GB upload limit
- Search for: `post_max_size` → change to `5120M`
- Search for: `memory_limit` → change to `1024M`  # ✅ 1GB Ram Allocated
- Search for: `max_execution_time` → change to `3600`
- Search for: `max_input_time` → change to `3600`

### Step 3: Restart PHP-FPM and Nginx   **[END OF PROCESS TO HOST ANOTHER WORDPRESS WEBSITE]**
```bash
sudo systemctl restart php8.2-fpm
sudo systemctl reload nginx                     
```

---

## **10. Install and Configure Cloudflared Tunnel [OPTIONAL]**

### Add Cloudflare GPG key
```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
```

### Add Cloudflare repo to APT sources
```bash
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
```

### Install cloudflared
```bash
sudo apt-get update && sudo apt-get install cloudflared
```

### Install Cloudflare Tunnel as a service
```bash
sudo cloudflared service install <TOKEN NUMBER>
```

### Run the Cloudflare Tunnel


### ✅ Enable Cloudflared Tunnel to Auto-Start on Boot

If you’ve already installed the Cloudflare Tunnel service using a token, but it doesn’t auto-start after reboot, follow these steps:

#### **Step 1: Verify the systemd service file**
Check if the service file exists:
```bash
sudo nano /etc/systemd/system/cloudflared.service
```

It should look like this (adjust the path/token if needed):

```ini
[Unit]
Description=cloudflared
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=0
Type=notify
ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel run --token <TOKEN NUMBER>
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

#### **Step 2: Reload systemd**
```bash
sudo systemctl daemon-reload
```

#### **Step 3: Enable the cloudflared service**
```bash
sudo systemctl enable cloudflared.service
```

#### **Step 4: Start the service**
```bash
sudo systemctl start cloudflared.service
```

#### **Step 5: Reboot and verify**
```bash
sudo reboot
```

Then confirm the service is running:
```bash
sudo systemctl status cloudflared.service
```

If everything is configured correctly, the tunnel will automatically start every time the server reboots. ✅

---

## **11. Complete WordPress Installation via Web Interface**

- Go to: `<WEBSITE DOMAIN>`
- Use the following database credentials:
  - **Username:** `<USER NAME>`
  - **Password:** `<PASSWORD>`

---
## 12. Post-Installation WordPress Checklist (Immediately After Setup)

Once your WordPress site is up and running via the web interface, follow these crucial steps to prepare it for design and production use:

---

### ✅ 1. Deactivate and Delete Default Plugins & Themes

* Go to **Plugins → Installed Plugins**

  * Deactivate and delete all default plugins.
* Go to **Appearance → Themes**

  * Delete all default themes except the one you plan to use (e.g., Astra).

---

### ✅ 2. Install Astra Theme

* Navigate to **Appearance → Themes → Add New**
* Search for `Astra`
* Click **Install** → **Activate**

---

### ✅ 3. Install Essential Plugins

Go to **Plugins → Add New** and install:

| Plugin Name       | Source                |
| ----------------- | --------------------- |
| Elementor         | WordPress Repo        |
| Elementor Pro     | Upload manually (ZIP) |
| Disable Gutenberg | WordPress Repo        |
| LiteSpeed Cache   | WordPress Repo        |
| WP Vivid Backup   | WordPress Repo        |
| MalCare           | WordPress Repo        |

* After installation, **activate all plugins**.
* For **Elementor Pro**, go to `Plugins → Add New → Upload Plugin`, upload the `.zip` file, and activate it.

---

### ✅ 4. Create Homepage

* Go to **Pages → Add New**
* Title it `Homepage`
* Publish the page

---

### ✅ 5. Set Homepage as Default Landing Page

* Go to **Settings → Reading**
* Set:

  * **Your homepage displays:** A static page
  * **Homepage:** `Homepage` (the one you just created)

---

### ✅ 6. Configure Permalinks

* Go to **Settings → Permalinks**
* Set to:

  ```
  Post name
  ```
* Save changes

---

## **13. Remove an Existing WordPress Website (Complete Cleanup Guide)**

Use this guide to completely remove an existing WordPress site from your Linux server, including its files, database, users, and NGINX config.

---



### ⚠️ Before You Begin

Double-check the website name, database name, and user before proceeding. This process is permanent and cannot be undone.

---


### 🔥 Step 1: Remove WordPress Files

```bash
sudo rm -rf /var/www/<WEBSITE_NAME>
```

---

### 🧾 Step 2: Delete NGINX Configuration

```bash
sudo rm /etc/nginx/sites-available/<WEBSITE_NAME>
sudo rm /etc/nginx/sites-enabled/<WEBSITE_NAME>
```

Reload NGINX to apply the changes:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### 💾 Step 3: Drop the Database and User from MariaDB

Access the MariaDB shell:

```bash
sudo mysql -u root -p
```

Then run:

```sql
DROP DATABASE db_<WEBSITE_NAME>;
DROP USER 'admin@<WEBSITE_NAME>'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

### 🔄 Step 4: Restart PHP and NGINX

```bash
sudo systemctl restart php8.2-fpm
sudo systemctl reload nginx
```

---

### 🧼 Done!

Your WordPress website, database, and all related configurations have been successfully removed from the server.
