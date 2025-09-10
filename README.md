# 🚀 NGINX + SSL Hosting with Certbot – Full Setup Guide

This guide walks you through setting up a secure NGINX web server with free SSL certificates from Let's Encrypt using Certbot. It includes steps to configure virtual hosts using `sites-available` and `sites-enabled`, enable HTTPS, and ensure auto-renewal of certificates.

---

## 📋 Prerequisites

- A Linux server (Ubuntu/Debian preferred) with root access
- A domain name pointed to your server's public IP (A record set)
- Application files (HTML/static or backend API)

---

## 🛠️ Step 1: Install NGINX

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

---

## 📁 Step 2: Create Your Web Directory

```bash
sudo mkdir -p /var/www/example.com
sudo chown -R $USER:$USER /var/www/example.com
sudo chmod -R 755 /var/www
echo "<h1>Hello from example.com!</h1>" | sudo tee /var/www/example.com/index.html
```

---

## ⚙️ Step 3: Add NGINX Site Configuration

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Paste the following config:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 🔗 Step 4: Enable the Configuration

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔐 Step 5: Install Certbot & Generate SSL

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Run Certbot to obtain and install SSL certificates:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Choose redirect option (Option 2) when prompted.

---

## 🔄 Step 6: Verify Auto-Renewal

Check if the systemd timer is active:

```bash
sudo systemctl status certbot.timer
```

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## 📃 Final NGINX Configuration (Post-SSL)

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    root /var/www/example.com;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 🧹 Optional Cleanup

Remove default site if unused:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

## 🛠️ Helpful Commands

| Task                   | Command                                     |
|------------------------|---------------------------------------------|
| Check NGINX config     | `sudo nginx -t`                             |
| Reload NGINX           | `sudo systemctl reload nginx`              |
| Run certbot manually   | `sudo certbot --nginx -d example.com`      |
| Test cert renew        | `sudo certbot renew --dry-run`             |
| Certbot logs           | `sudo journalctl -u certbot.timer`         |

---

## ✅ Done!

Your site is now:

- Hosted using **NGINX**
- Secured with **HTTPS**
- Automatically renewed via **Certbot**
- Managed with clean virtual host files (`sites-available`, `sites-enabled`)

---

> ✍️ Replace `example.com` with your actual domain in the above commands and config files.

Need help with reverse proxying backend apps or wildcard domains? Just ping me!


---

## 🔁 Hosting Applications on Localhost with Reverse Proxy

If you're running applications on your server (e.g., `localhost:5000` for frontend and `localhost:8000` for backend), use NGINX as a reverse proxy to direct traffic.

---

### 🌐 Scenario 1: Serve `/` from `localhost:5000` and `/api` from `localhost:8000`

**NGINX Configuration:**

```nginx
server {
    listen 80;
    server_name example.com;
    access_log  /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log;

    proxy_headers_hash_max_size 512;
    proxy_headers_hash_bucket_size 128;
	client_max_body_size 50M;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # Add these lines:
        proxy_connect_timeout 600s;
        proxy_send_timeout    600s;
        proxy_read_timeout    600s;
        send_timeout          600s;

        # Add these buffer size settings to fix header issue
        proxy_buffer_size          16k;
        proxy_buffers              8 16k;
        proxy_busy_buffers_size    32k;

    }

    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Now requests to:

- `http://example.com/` → go to `localhost:5000`
- `http://example.com/api/...` → go to `localhost:8000/...`

---

### 🔄 Scenario 2: Remove `/api` prefix and forward to `localhost:8000`

**Example:**

- Incoming: `http://example.com/api/users`
- Forward to: `http://localhost:8000/users`

Update the `location` block:

```nginx
location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://localhost:8000/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
}
```

This strips the `/api` prefix and forwards the rest to the backend.

---

### 🧪 Test Everything

After saving your config:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

> 🔁 Using reverse proxy keeps your ports hidden and enables clean URLs. Make sure your backend services accept connections from `localhost`.

