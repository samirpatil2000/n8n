# N8N + PostgreSQL + NGINX Setup

This documentation explains how to set up **n8n** with **PostgreSQL** using Docker Compose, and expose it securely through **NGINX** with SSL termination.

---

## 1. Requirements

* **Ubuntu server** with Docker & Docker Compose installed
* **Domain name**: `n8n.your-company.tdl` pointing to your server
* **SSL certificates** for HTTPS termination

---

## 2. NGINX Reverse Proxy Configuration

Create `/etc/nginx/sites-available/n8n.conf`:

```nginx
server {
    listen 443 ssl http2;
    client_max_body_size 2G;
    ssl_certificate /etc/nginx/ssl/ssl-bundle.crt;
    ssl_certificate_key /etc/nginx/ssl/sandbox_private_key.pem;

    server_name n8n.your-company.tdl;

    proxy_read_timeout 300;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy "strict-origin";

    location / {
        proxy_pass http://127.0.0.1:5678/;

        # Required for WebSockets support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
```

Then enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

---

## 3. Environment Variables

Create a `.env` file:

```dotenv
# Database Configuration
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=postgres
DB_POSTGRESDB_PASSWORD=postgres

# N8N Configuration
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=secret

N8N_HOST=n8n.your-company.tdl
N8N_PROTOCOL=https
N8N_PORT=5678

VUE_APP_URL_BASE_API=https://n8n.your-company.tdl/
```

---

## 4. Docker Compose File

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: ${DB_POSTGRESDB_USER}
      POSTGRES_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      POSTGRES_DB: ${DB_POSTGRESDB_DATABASE}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432" # Optional: Expose if you need local access

  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    env_file:
      - .env
    depends_on:
      - postgres
    volumes:
      - ./n8n_data:/home/node/.n8n

volumes:
  postgres_data:
  n8n_data:
```

---

## 5. Start the Services

```bash
docker-compose up -d
```

Check logs:

```bash
docker-compose logs -f n8n
```

---

## 6. Access the Instance

Open your browser and visit:

**➡️ [https://n8n.your-company.tdl](https://n8n.your-company.tdl)**

Login using credentials from `.env`:

* **Username:** `admin`
* **Password:** `secret`

---

## 7. Tips

* Use a strong password in production for `N8N_BASIC_AUTH_PASSWORD`
* Ensure firewall allows ports **80** and **443**
* WebSocket support is required for real-time updates in n8n, so keep `proxy_http_version`, `Upgrade`, and `Connection` headers in your NGINX configuration.
