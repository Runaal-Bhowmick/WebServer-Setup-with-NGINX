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
sudo mysql_secure_installation   # Set root password as 'password'
```

### Access MariaDB and run the following:  **[START FROM HERE TO HOST ANOTHER WORDPRESS WEBSITE]**
```bash
sudo mysql -u root -p
```

### Inside the MariaDB shell:
```sql
CREATE DATABASE <DATABASE NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER '<USER NAME>'@'localhost' IDENTIFIED BY '<PASSWORD>';
GRANT ALL PRIVILEGES ON <DATABASE NAME>.* TO '<USER NAME>'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

## **4. Install PHP and phpMyAdmin**
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
sudo chown -R www-data:www-data wordpress/
sudo chmod -R 755 wordpress/
sudo rm -r latest.tar.gz
cd wordpress
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

- Change the **Database Name**, **User Name**, and **Password**.
- Add salts from: [https://api.wordpress.org/secret-key/1.1/salt/](https://api.wordpress.org/secret-key/1.1/salt/)

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
    listen 80;
    server_name <WEBSITE DOMAIN>;

    root /var/www/<WEBSITE NAME>/wordpress;
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
sudo systemctl restart nginx     # [END OF PROCESS TO HOST ANOTHER WORDPRESS WEBSITE]
```

---

## **7. Install and Configure Cloudflared Tunnel [OPTIONAL]**

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
```bash
cloudflared tunnel run --token <TOKEN NUMBER>
```

---

## **8. Complete WordPress Installation via Web Interface**

- Go to: `<WEBSITE DOMAIN>`
- Use the following database credentials:
  - **Database Name:** `<DATABASE NAME>`
  - **Username:** `<USER NAME>`
  - **Password:** `<PASSWORD>`
