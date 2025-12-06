# n8n + WAHA Deployment Guide

## Prerequisites
- VPS with Ubuntu (Hostinger)
- 2 subdomains pointed to VPS IP

## Files Overview
- `.env` - General config (domains, timezone)
- `.waha.env` - WAHA-specific config (credentials, webhooks)
- `docker-compose.yml` - Service definitions
- `nginx/conf.d/default.conf` - Reverse proxy config

---

## 1. DNS Setup (Hostinger)
Add A records in DNS zone:
```
n8n.yourdomain.com  → YOUR_VPS_IP
waha.yourdomain.com → YOUR_VPS_IP
```
Wait 5-10min for propagation.

---

## 2. SSH & System Setup
```bash
ssh root@YOUR_VPS_IP
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin -y
```

---

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

---

## 4. Project Setup
```bash
mkdir -p /opt/automation && cd /opt/automation
mkdir -p nginx/conf.d certbot/www certbot/conf
```

Upload files to `/opt/automation/`:
- `docker-compose.yml`
- `.env.example`
- `.waha.env.example`

---

## 5. Configure Environment Files

### Main .env
```bash
cp .env.example .env
nano .env
```
Update with your actual domains.

### WAHA .env (Option A: Auto-generate credentials)
```bash
# Create base file
cp .waha.env.example .waha.env

# Generate secure credentials using WAHA's init script
docker run --rm -v "$(pwd)":/app/env devlikeapro/waha init-waha /app/env .waha.env .waha.env --force

# Edit to add your domains/webhooks
nano .waha.env
```

### WAHA .env (Option B: Manual setup)
```bash
cp .waha.env.example .waha.env
nano .waha.env
```

Generate secure random keys:
```bash
# Generate 32-char random strings for passwords/keys
uuidgen | tr -d '-'
```

Update these values in `.waha.env`:
- `WAHA_DASHBOARD_PASSWORD` - random 32 chars
- `WAHA_API_KEY` - random 32 chars
- `WAHA_BASE_URL` - `https://waha.yourdomain.com`
- `WHATSAPP_HOOK_URL` - `https://n8n.yourdomain.com/webhook/waha`

---

## 6. Create Nginx Config

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

---

## 7. Get SSL Certificates

Create temp HTTP-only nginx config:
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

# Get n8n cert (replace domain and email)
docker run --rm \
  -v $(pwd)/certbot/www:/var/www/certbot \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  --webroot-path=/var/www/certbot \
  -d n8n.yourdomain.com \
  --email your@email.com --agree-tos --no-eff-email

# Get waha cert (replace domain and email)
docker run --rm \
  -v $(pwd)/certbot/www:/var/www/certbot \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  certbot/certbot certonly --webroot \
  --webroot-path=/var/www/certbot \
  -d waha.yourdomain.com \
  --email your@email.com --agree-tos --no-eff-email
```

---

## 8. Deploy

Replace nginx config with full HTTPS version (from step 6), then:
```bash
docker compose down
docker compose up -d
docker compose ps
docker compose logs -f
```

---

## 9. Access & Verify

| Service | URL | Auth |
|---------|-----|------|
| n8n | https://n8n.yourdomain.com | Create account on first visit |
| WAHA Dashboard | https://waha.yourdomain.com | Username/password from `.waha.env` |
| WAHA Swagger | https://waha.yourdomain.com/api/docs | Same as dashboard |

---

## 10. Connect WhatsApp

1. Open WAHA Dashboard: https://waha.yourdomain.com
2. Login with credentials from `.waha.env`
3. Create a new session (or use Swagger: `POST /api/sessions`)
4. Scan QR code with WhatsApp mobile app
5. Session status should change to `WORKING`

---

## 11. Create n8n Webhook

1. Open n8n: https://n8n.yourdomain.com
2. Create new workflow
3. Add **Webhook** node:
   - HTTP Method: `POST`
   - Path: `waha`
4. Activate the workflow
5. Test by sending a WhatsApp message - it should appear in n8n

---

## Useful Commands

```bash
# View logs
docker compose logs -f
docker compose logs -f waha
docker compose logs -f n8n

# Restart services
docker compose restart

# Stop all
docker compose down

# Update images
docker compose pull && docker compose up -d

# Check WAHA health
curl -H "X-Api-Key: YOUR_API_KEY" https://waha.yourdomain.com/api/health
```

---

## Troubleshooting

### WAHA won't start
```bash
docker compose logs waha
# Check .waha.env syntax
```

### Webhook not receiving events
1. Verify `WHATSAPP_HOOK_URL` in `.waha.env` is correct
2. Check n8n workflow is activated
3. Test webhook URL is accessible from WAHA container:
   ```bash
   docker compose exec waha curl -I https://n8n.yourdomain.com/webhook/waha
   ```

### SSL certificate issues
```bash
# Check cert exists
ls -la certbot/conf/live/

# Renew manually
docker run --rm -v $(pwd)/certbot/www:/var/www/certbot -v $(pwd)/certbot/conf:/etc/letsencrypt certbot/certbot renew
```
