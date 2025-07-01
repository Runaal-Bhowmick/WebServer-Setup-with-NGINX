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
CREATE USER 'admin@<WEBSITE NAME>'@'localhost' IDENTIFIED BY '<PASSWORD>';
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
sudo mv wordpress public_html   # ✅ Rename folder
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
upstream <WEBSITE NAME>-php-handler {
    server unix:/var/run/php/php8.2-fpm.sock;
}

server {
    client_max_body_size 5120M; # ✅ 5GB upload limit
    listen 80;
    server_name <WEBSITE DOMAIN>;

    root /var/www/<WEBSITE NAME>/public_html; 
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass <WEBSITE NAME>-php-handler;
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
```

### Then run:
```bash
sudo ln -s /etc/nginx/sites-available/<WEBSITE NAME> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx     
```

---

## **7. Increase Max File Upload Limit to 10GB**

To allow large uploads (up to 10 GB), modify PHP configuration.

### Step 1: Edit `php.ini` for PHP-FPM
```bash
sudo nano /etc/php/8.2/fpm/php.ini
```

### In the file, search using `Ctrl + W` in nano:
- Search for: `upload_max_filesize` → change to `5120M` # ✅ 5GB upload limit
- Search for: `post_max_size` → change to `5120M`
- Search for: `memory_limit` → change to `5120M`
- Search for: `max_execution_time` → change to `3600`
- Search for: `max_input_time` → change to `3600`

### Step 3: Restart PHP-FPM and Nginx   **[END OF PROCESS TO HOST ANOTHER WORDPRESS WEBSITE]**
```bash
sudo systemctl restart php8.2-fpm
sudo systemctl reload nginx                     
```

---

## **8. Install and Configure Cloudflared Tunnel [OPTIONAL]**

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

## **9. Complete WordPress Installation via Web Interface**

- Go to: `<WEBSITE DOMAIN>`
- Use the following database credentials:
  - **Username:** `<USER NAME>`
  - **Password:** `<PASSWORD>`

---

## **10. Install Recommended WordPress Plugin (Before Doing Anything Else)**

### ✅ WPVivid: For Auto Google Drive Backups and Site Migration

This plugin is highly recommended for managing site backups and performing migrations easily.

- **Official WordPress Download**: [WPVivid Plugin - WordPress.org](https://wordpress.org/plugins/wpvivid-backuprestore/)
- **Direct GDrive Download (v0.9.116)**: [Download from Google Drive](https://drive.google.com/file/d/1IMnF4h-Jf-KCZgFIy435GyHcM24pfj0Q/view?usp=drive_link)
