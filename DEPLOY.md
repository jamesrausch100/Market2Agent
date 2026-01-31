# Market2Agent - Deployment Guide

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│  DROPLET 1 (10.120.0.2) - Web Frontend                         │
│  market2agent.com                                              │
│  - Static HTML + JS                                            │
│  - Nginx serving frontend                                      │
│  - Calls API on Droplet 2                                      │
└───────────────────────────┬────────────────────────────────────┘
                            │ HTTPS
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  DROPLET 2 (10.120.0.4) - Audit Engine                         │
│  api.market2agent.com                                          │
│  - FastAPI application                                         │
│  - arq workers (async job processing)                          │
│  - Redis (job queue)                                           │
│  - Crawlers + Analyzers                                        │
└───────────────────────────┬────────────────────────────────────┘
                            │ Bolt (7687)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  DROPLET 3 (10.120.0.3) - Graph Database                       │
│  Internal only - no public access                              │
│  - Neo4j Community Edition                                     │
│  - Stores domains, entities, audits, relationships             │
└────────────────────────────────────────────────────────────────┘
```

---

## Deployment Steps

### Step 1: Set up Droplet 3 (Neo4j)

SSH into Droplet 3:
```bash
ssh root@<droplet3-public-ip>
```

Upload and run the setup script:
```bash
# Copy droplet3_neo4j_setup.sh to server, then:
chmod +x droplet3_neo4j_setup.sh
./droplet3_neo4j_setup.sh
```

Initialize the schema:
```bash
cypher-shell -u neo4j -p M2A_graph_2024_secure -f neo4j_schema.cypher
```

Verify Neo4j is running:
```bash
curl http://10.120.0.3:7474
```

---

### Step 2: Set up Droplet 2 (Audit Engine)

SSH into Droplet 2:
```bash
ssh root@<droplet2-public-ip>
```

Run the setup script:
```bash
chmod +x droplet2_audit_engine_setup.sh
./droplet2_audit_engine_setup.sh
```

Copy the application code:
```bash
# From your local machine or this repo:
scp -r audit_engine/app/* root@<droplet2-ip>:/opt/market2agent/app/
scp audit_engine/.env.example root@<droplet2-ip>:/opt/market2agent/.env
```

Start the services:
```bash
systemctl daemon-reload
systemctl enable m2a-api m2a-worker
systemctl start m2a-api m2a-worker
```

Verify API is running:
```bash
curl http://10.120.0.4:8000/health
```

---

### Step 3: Set up Droplet 1 (Web Frontend)

SSH into Droplet 1:
```bash
ssh root@<droplet1-public-ip>
```

Install Nginx and set up the frontend:
```bash
apt-get update && apt-get install -y nginx

# Copy frontend files
mkdir -p /var/www/market2agent
cp web_frontend/index.html /var/www/market2agent/

# Configure Nginx
cat > /etc/nginx/sites-available/market2agent << 'EOF'
server {
    listen 80;
    server_name market2agent.com www.market2agent.com;
    root /var/www/market2agent;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

ln -sf /etc/nginx/sites-available/market2agent /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

---

### Step 4: DNS Configuration

In your DNS provider (or Digital Ocean DNS):

| Type  | Name                | Value                    |
|-------|---------------------|--------------------------|
| A     | market2agent.com    | <droplet1-public-ip>     |
| A     | www                 | <droplet1-public-ip>     |
| A     | api                 | <droplet2-public-ip>     |

---

### Step 5: SSL (Let's Encrypt)

On Droplet 1:
```bash
apt-get install -y certbot python3-certbot-nginx
certbot --nginx -d market2agent.com -d www.market2agent.com
```

On Droplet 2:
```bash
apt-get install -y certbot python3-certbot-nginx
certbot --nginx -d api.market2agent.com
```

---

## API Endpoints

| Method | Endpoint                    | Description                     |
|--------|-----------------------------|---------------------------------|
| GET    | /health                     | Health check                    |
| POST   | /v1/audit                   | Request new audit               |
| GET    | /v1/audit/{audit_id}        | Get audit results               |
| GET    | /v1/domain/{domain}/history | Get audit history for domain    |
| GET    | /v1/leaderboard             | Top-scoring domains             |

---

## What the Audit Actually Measures

### Structured Data Score (35 points max)
- JSON-LD presence: 15 points
- Organization schema: 10 points
- WebSite schema: 5 points
- sameAs links: 5 points
- Minus points for validation errors

### Entity Presence Score (30 points max)
- Wikidata entry: 15 points
- Wikidata sameAs links: 5 points
- Wikipedia article: 10 points

### Content Clarity Score (20 points max)
- Clear page title: 5 points
- Meta description: 5 points
- Canonical URL: 3 points
- Organization description in structured data: 7 points

### Technical Signals Score (15 points max)
- MX records: 3 points
- SPF: 3 points
- DMARC: 4 points
- Basic crawlability: 5 points

---

## Testing

Test the full flow locally before deploying:

```bash
cd audit_engine

# Create venv
python3.11 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn httpx beautifulsoup4 lxml extruct neo4j redis arq pydantic python-dotenv structlog

# Run a test audit directly (requires Neo4j running)
python -m app.worker example.com
```

---

## Next Steps (Roadmap)

1. **Deploy to all 3 droplets** ← You are here
2. **Set up DNS and SSL**
3. **Test with real domains**
4. **Add user authentication** (for paid tiers)
5. **Add Stripe integration** (for payments)
6. **Add scheduled audits** (cron job triggering audits for subscribed domains)
7. **Add AI retrieval testing** (the "spicy" feature - actually query LLMs)

---

## File Manifest

```
/home/claude/
├── DEPLOY.md                        # This file
├── droplet3_neo4j_setup.sh          # Neo4j installation script
├── neo4j_schema.cypher              # Graph database schema
├── droplet2_audit_engine_setup.sh   # Audit engine setup script
├── audit_engine/
│   ├── .env.example                 # Environment variables
│   └── app/
│       ├── __init__.py
│       ├── main.py                  # FastAPI application
│       ├── db.py                    # Neo4j connection
│       ├── worker.py                # Async job worker
│       ├── crawlers/
│       │   ├── __init__.py
│       │   ├── structured_data.py   # JSON-LD/Schema extraction
│       │   └── entity_presence.py   # Wikidata/Wikipedia checks
│       └── analyzers/
│           ├── __init__.py
│           └── scoring.py           # GEO score calculation
└── web_frontend/
    └── index.html                   # Landing page with real API calls
```

---

## Security Notes

- Neo4j (Droplet 3) is only accessible via internal VPC IP - no public exposure
- Change the default Neo4j password in production
- Enable DO firewall to restrict public access:
  - Droplet 1: Allow 80, 443 from anywhere
  - Droplet 2: Allow 80, 443 from anywhere (for API)
  - Droplet 3: Block all public access, only allow 7687 from 10.120.0.4
