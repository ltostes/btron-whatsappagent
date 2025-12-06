# n8n + WAHA Deployment Guide

## Prerequisites
- VPS with Ubuntu (Hostinger)
- 2 subdomains pointed to VPS IP

## 1. DNS Setup (Hostinger)
Add A records in DNS zone:
```
n8n.yourdomain.com  → YOUR_VPS_IP
waha.yourdomain.com → YOUR_VPS_IP
```
Wait 5-10min for propagation.

## 2. SSH & System Setup
```bash
ssh root@YOUR_VPS_IP
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin -y
```

## 3. Firewall
```bash
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

## 4. Project Setup
```bash
mkdir -p /opt/automation && cd /opt/automation
mkdir -p nginx/conf.d certbot/www certbot/conf
```

Upload `docker-compose.yml` and `.env.example` to `/opt/automation/`.

```bash
cp .env.example .env
nano .env  # Edit with your actual values
```

## 5. Create Nginx Config

Create `nginx/conf.d/default.conf`:

```nginx
server {
    listen 80;
    server_name n8n.yourdomain.com waha.yourdomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name n8n.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/n8n.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    server_name waha.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/waha.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/waha.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://waha:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 6. Get SSL Certificates

First, create temp HTTP-only config:
```bash
cat > nginx/conf.d/default.conf << 'EOF'
server {
    listen 80;
    server_name n8n.yourdomain.com waha.yourdomain.com;
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}
EOF
```

Start nginx and get certs:
```bash
docker compose up -d nginx

# Get n8n cert
docker run --rm \
  -v $(pwd)/certbot/www:/var/www/certbot \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  --webroot-path=/var/www/certbot \
  -d n8n.yourdomain.com \
  --email your@email.com --agree-tos --no-eff-email

# Get waha cert
docker run --rm \
  -v $(pwd)/certbot/www:/var/www/certbot \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  --webroot-path=/var/www/certbot \
  -d waha.yourdomain.com \
  --email your@email.com --agree-tos --no-eff-email
```

## 7. Deploy

Replace nginx config with full HTTPS version (from step 5), then:
```bash
docker compose down
docker compose up -d
docker compose ps
docker compose logs -f
```

## 8. Access

- **n8n**: https://n8n.yourdomain.com (create account on first visit)
- **WAHA**: https://waha.yourdomain.com/api/docs (Swagger UI)

## 9. Connect WAHA to n8n

1. In WAHA Swagger UI, create a session: `POST /api/sessions`
2. Scan QR code to link WhatsApp
3. In n8n, create Webhook node with path `/webhook/waha`
4. Messages will flow from WhatsApp → WAHA → n8n

## Useful Commands

```bash
# View logs
docker compose logs -f

# Restart services
docker compose restart

# Stop all
docker compose down

# Update images
docker compose pull && docker compose up -d
```
