# SOP: Web Server Setup and Usage (Nginx & Apache)

**Scope:** Linux-based servers (Ubuntu/Debian)  
**Applies to:** Nginx, Apache2

---

## 1. Installation

### Nginx
```bash
sudo apt update && sudo apt install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Apache
```bash
sudo apt update && sudo apt install apache2
sudo systemctl start apache2
sudo systemctl enable apache2
```

Verify by visiting `http://localhost` — default welcome page should appear.

---

## 2. Create a Virtual Host

### Nginx

Create `/etc/nginx/sites-available/mysite.conf`:

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Apache

Create `/etc/apache2/sites-available/mysite.conf`:

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/mysite

    <Directory /var/www/mysite>
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

Enable and reload:

```bash
sudo a2ensite mysite.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

---

## 3. Reverse Proxy (Nginx)

Use Nginx to proxy dynamic requests to an app (Node, Python, etc.) while serving static files directly:

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        root /var/www/myapp;
        expires 30d;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 4. Enable HTTPS (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx   # for Nginx
sudo apt install certbot python3-certbot-apache  # for Apache

sudo certbot --nginx -d example.com   # Nginx
sudo certbot --apache -d example.com  # Apache
```

Certificates auto-renew via a cron job installed by Certbot.

---

## 5. Test and Reload Config

| Action | Nginx | Apache |
|---|---|---|
| Test config syntax | `sudo nginx -t` | `sudo apache2ctl configtest` |
| Reload (no downtime) | `sudo systemctl reload nginx` | `sudo systemctl reload apache2` |
| Full restart | `sudo systemctl restart nginx` | `sudo systemctl restart apache2` |

> Always run the config test before reloading in production.

---

## 6. Common File Locations

| Item | Nginx | Apache |
|---|---|---|
| Main config | `/etc/nginx/nginx.conf` | `/etc/apache2/apache2.conf` |
| Site configs | `/etc/nginx/sites-available/` | `/etc/apache2/sites-available/` |
| Enabled sites | `/etc/nginx/sites-enabled/` | `/etc/apache2/sites-enabled/` |
| Access log | `/var/log/nginx/access.log` | `/var/log/apache2/access.log` |
| Error log | `/var/log/nginx/error.log` | `/var/log/apache2/error.log` |
| Web root | `/var/www/html` | `/var/www/html` |

---

## 7. When to Use Which

| Use case | Recommended |
|---|---|
| Reverse proxy / high concurrency | Nginx |
| Static file serving | Nginx |
| Shared hosting / `.htaccess` support | Apache |
| Legacy PHP apps (WordPress, Laravel) | Apache |
| SSL termination front-end | Nginx |
