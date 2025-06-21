# 🚀 DockNcrypt: Automating HTTPS with Docker, Nginx & Certbot

For any web-based project — production, demo, or something temporary like a college event ([Network Treasure Hunt](https://nth.samirwankhede.in/)) — setting up HTTPS is essential. It avoids scary browser warnings, ensures trust, and protects user data.

But let's be honest: manually configuring SSL with Nginx and Let’s Encrypt is tedious. So I **automated** it with:

- 🐳 Dockerized Nginx
- 🔒 Certbot (Let’s Encrypt)
- 🧩 One-command reproducible setup

```bash
docker compose up
```

This command sets up reverse proxy, SSL, backend, and static content — securely, automatically.

For Proper Component and parts explain like I'm 5 version checkout: [Workflow Blog](https://medium.com/fossible/automating-https-with-docker-nginx-certbot-c4c406409f32)

---

## 📦 Project Structure

```
├── backend/           # Node.js backend API
├── nginx/             # Main HTTPS Nginx reverse proxy
├── nginx-certbot/     # Temporary server for ACME challenge
└── docker-compose.yml # Orchestrates everything
```

Each component is modular for easy maintenance and scaling.

---

## 🧠 Backend – `backend/`

A basic Node.js app running on port `3000`.

```
├── backend/
│   ├── Dockerfile
│   ├── index.js
│   └── package.json
```

**Dockerfile:**
```Dockerfile
FROM node:20.18.2-alpine3.21

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

---

## 🌐 Temporary HTTP Nginx – `nginx-certbot/`

Handles Let's Encrypt HTTP-01 challenge.

```
├── nginx-certbot/
│   ├── Dockerfile
│   └── nginx-certbot.conf
```

**nginx-certbot.conf:**
```nginx
server {
    listen 80;
    server_name autonginx.samirwankhede.in;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        try_files $uri =404;
    }

    location / {
        return 404;
    }
}
```

**Dockerfile:**
```Dockerfile
FROM nginx:latest

COPY nginx-certbot.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## 🔐 Nginx HTTPS Reverse Proxy – `nginx/`

Serves static files, proxies backend, enforces HTTPS, and blocks bad bots.

```
├── nginx/
│   ├── Dockerfile
│   ├── html/
│   └── nginx.conf
```

**nginx.conf (HTTPS & Security focused):**
```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    limit_req_zone $binary_remote_addr zone=api_limit:20m rate=10r/s;

    map $http_user_agent $block_bad_user_agent {
        default 0;
        "~*curl" 1;
        "~*wget" 1;
        "~*python-requests" 1;
        "~*axios" 1;
        "~*PostmanRuntime" 1;
    }

    server {
        listen 80;
        server_name autonginx.samirwankhede.in;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name autonginx.samirwankhede.in;

        ssl_certificate /etc/letsencrypt/live/autonginx.samirwankhede.in/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/autonginx.samirwankhede.in/privkey.pem;

        if ($block_bad_user_agent) {
            return 403;
        }

        location /backend/server/ {
            limit_req zone=api_limit burst=20 nodelay;
            proxy_pass http://backend:3000/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Cookie $http_cookie;
        }

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /rickroll-rick.gif {
            root /usr/share/nginx/html;
        }

        error_page 404 /404.html;
        location = /404.html {
            root /usr/share/nginx/html;
            internal;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
            internal;
        }
    }
}
```

**Dockerfile:**
```Dockerfile
FROM nginx:latest

COPY html /usr/share/nginx/html

COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

---

## 🛠️ Docker Compose Orchestrator

```yaml
services:

  nginx-certbot:
    build: ./nginx-certbot
    volumes:
      - certbot_challenges:/var/www/certbot
    ports:
      - "80:80"

  certbot:
    image: certbot/certbot
    volumes:
      - letsencrypt:/etc/letsencrypt
      - certbot_challenges:/var/www/certbot
    entrypoint: >
      sh -c "certbot certonly --webroot --webroot-path=/var/www/certbot 
      --email some@gmail.com --agree-tos --no-eff-email 
      --keep-until-expiring -d autonginx.samirwankhede.in && echo '✅ Certbot finished successfully!'"
    depends_on:
      nginx-certbot:
        condition: service_started

  shutdown-nginx-certbot:
    image: alpine
    depends_on:
      certbot:
        condition: service_completed_successfully
    command: >
      sh -c "echo 'Certbot finished, shutting down nginx-certbot...' && 
      sleep 2 && kill -TERM 1"
    pid: "service:nginx-certbot"

  backend:
    build: ./backend
    ports: 
      - "3000:3000"
    restart: unless-stopped

  nginx:
    build: ./nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - letsencrypt:/etc/letsencrypt
      - certbot_challenges:/var/www/certbot
    depends_on:
      backend:
        condition: service_started
      shutdown-nginx-certbot:
        condition: service_completed_successfully
    restart: always

volumes:
  letsencrypt:
  certbot_challenges:
```

---

## ⚠️ Things to Always Check

- ✅ DNS A record points to your server
- ✅ Only one container uses ports 80/443 at a time
- ✅ Certbot rate limits apply – avoid retries
- ✅ Your cloud firewall/security group allows 80 & 443

---

## 📸 Demo

![Demo 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bedxeono0f9y4qvfitqy.png)
![Demo 2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m3yc3iq6nuu0z9b7le56.png)
![Demo 3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/06yw3ffgybapoul5pmqd.png)
![Demo 4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ljhiv94kk1j5uslks0jz.png)
![Demo 5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/klx3go1uvjn3ja138btt.png)
![Demo 6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/at80khsz4vuog5271gv0.png)

---

## 🤝 Contribute

Whether you're:

- Hosting tech events like NTH
- Learning DevOps
- Building secure infra for the first time

You’re welcome to **fork**, **clone**, or **open PRs**.

Even fixing typos or adding docs helps others. Ideas? Add issues or reach out!

---

## 🌐 Connect

**Project Repo** 👉 [github.com/DockNcrypt/DockNcrypt](https://github.com/DockNcrypt/DockNcrypt)

**Creator** 👉 [samirwankhede.in](https://samirwankhede.in)

**Python Package** 👉 [github.com/DockNcrypt/DockNcrypt-Python-Package](https://github.com/DockNcrypt/DockNcrypt-Python-Package)

---
```dockerfile
# dockNcrypt ⚓ - Secure automation of SSL for Nginx via Docker and Certbot
```
