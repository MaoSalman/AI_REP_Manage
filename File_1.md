PART 1: TECHNICAL INFRASTRUCTURE (Copy-Paste Ready)

Step 2.1: Server Setup (DigitalOcean)
Sign up: digitalocean.com (use referral for $200 credit)
Create Droplet:
Click "Create" → "Droplets"
Choose Region: Frankfurt (EU, GDPR compliant)
Choose Image: Ubuntu 24.04 (LTS)
Choose Plan: Basic $12/month (2GB RAM / 1 CPU) — enough for 50+ clients
Authentication: SSH Key (generate below)
Hostname: review-agent-prod
Click "Create"
Generate SSH Key (on your computer):
bash
# Mac/Linux — run in Terminal
ssh-keygen -t ed25519 -C "your-email@example.com"
# Press Enter to save to default location
# Enter passphrase (or leave empty for convenience)
cat ~/.ssh/id_ed25519.pub
# Copy the output and paste into DigitalOcean SSH key field

# Windows — run in PowerShell
ssh-keygen -t ed25519 -C "your-email@example.com"
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
# Copy output
Connect to Server:
bash
ssh root@YOUR_DROPLET_IP

Step 2.2: Install Docker & Docker Compose
bash
# Run ALL of these commands on your server (copy-paste entire block)

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Add user to docker group
usermod -aG docker $USER
newgrp docker

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/download/v2.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Verify
docker --version
docker-compose --version

# Install basic tools
apt install -y git nginx certbot python3-certbot-nginx ufw

# Setup firewall
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable

Step 2.3: Install n8n (Self-Hosted)
bash
# Create directory
mkdir -p /opt/n8n && cd /opt/n8n

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=YourStrongPassword123!
      - N8N_HOST=automation.yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://automation.yourdomain.com/
      - GENERIC_TIMEZONE=Europe/Rome
      - EXECUTIONS_MODE=regular
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_SAVE_ON_PROGRESS=true
      - EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS=true
      - DB_TYPE=sqlite
      - N8N_ENCRYPTION_KEY=YourRandomEncryptionKeyChangeThis123456789
    volumes:
      - ~/.n8n:/home/node/.n8n
      - /opt/n8n/backups:/backups

  # Optional: PostgreSQL for better performance at scale
  postgres:
    image: postgres:16-alpine
    restart: always
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=n8nPassword123!
      - POSTGRES_DB=n8n
    volumes:
      - /opt/n8n/postgres-data:/var/lib/postgresql/data
    ports:
      - "127.0.0.1:5432:5432"
EOF

# Start n8n
docker-compose up -d

# Check logs
docker-compose logs -f n8n
Wait 2 minutes, then access: http://YOUR_SERVER_IP:5678
Login: admin / YourStrongPassword123!

Step 2.4: Setup Domain & SSL (Nginx Reverse Proxy)
Buy domain: Namecheap.com or Cloudflare (€10–€15/year)
Point DNS A record: automation.yourdomain.com → YOUR_DROPLET_IP
bash
# Create Nginx config
cat > /etc/nginx/sites-available/automation << 'EOF'
server {
    listen 80;
    server_name automation.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF

# Enable site
ln -s /etc/nginx/sites-available/automation /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx

# Get SSL certificate
certbot --nginx -d automation.yourdomain.com

# Auto-renewal is set up automatically
n8n is now live at: https://automation.yourdomain.com

Step 2.5: Setup Airtable (Database)
Go to airtable.com → Sign up (free plan: 1,000 records/base)
Create Base: "Review Agent Clients"
Create Table: "Clients"
Fields to create:
Table
Field Name	Type	Options
Client Name	Single line text	
Business Type	Single select	Restaurant, Dental, Law, Real Estate, Other
Google Place ID	Single line text	
Yelp Business ID	Single line text	
Status	Single select	Active, Setup, Paused, Cancelled
Monthly Fee	Currency	EUR
Setup Fee Paid	Checkbox	
Review Response Tone	Long text	"Professional, friendly, mention our commitment to service"
Owner Email	Email	
Phone	Phone	
Created	Created time	
Last Review Checked	Date	
Get Airtable API Key:
Go to airtable.com/create/tokens
Create token with scopes: data.records:read, data.records:write, schema.bases:read
Note the Base ID (from URL) and Token

Step 2.6: Setup OpenAI API
Go to platform.openai.com
Add payment method (credit card)
Go to API Keys → Create new secret key
Copy and save immediately (won't show again)
Recommended model for review responses: gpt-4.1 ($0.10 input / $0.40 output per 1M tokens) 
Cost estimate: 1 review response = ~500 tokens = $0.00025. For 100 reviews/day = $0.025/day = ~$0.75/month per client.

Step 2.7: Setup Google Cloud (for Reviews API)
You need Google Places API to monitor reviews.
Go to console.cloud.google.com
Create new project: "review-agent"
Enable APIs:
Places API (New)
Google Places API
Create API Key (restrict to Places API only)
Add billing (required, but free tier: 5,000 requests/day)