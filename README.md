# Multi-App Docker Infrastructure

This setup manages multiple applications (BigCapital, Corteza, etc.) on a single machine with different hostnames.

## Structure

```
docker-infrastructure/
├── README.md
├── docker-compose.yml          # Main orchestration file
├── .env.example                # Environment variables template
├── nginx/
│   ├── nginx.conf              # Master Nginx configuration (hostname-based routing)
│   └── conf.d/                 # Additional config snippets (optional)
├── apps/
│   ├── bigcapital/
│   │   ├── docker-compose.override.yml
│   │   ├── .env
│   │   └── services/           # Service-specific config
│   ├── corteza/
│   │   ├── docker-compose.override.yml
│   │   ├── .env
│   │   └── services/
│   └── [future-apps]/
└── shared/
    ├── certbot/                # SSL certificates
    └── volumes/                # Persistent data
```

## Hostnames

- `accounting.theworklabs.cc` → BigCapital
- `crm.theworklabs.cc` → Corteza CRM
- `crm.theworklabs.cc/theworklabs-docs/` → Corteza Docs (sub-path)

## How It Works

1. **Single Nginx instance** on port 80/443 routes based on hostname
2. **Docker Compose** orchestrates all services on one network
3. **Environment files** per app for isolated configuration
4. **Easy scaling** - add new apps without modifying main files

## Local Testing Setup (No SSL)

To test locally on your Mac without SSL certificates:

1. **Add hostnames to /etc/hosts**:
   ```bash
   sudo nano /etc/hosts
   # Add these lines:
   127.0.0.1 accounting.theworklabs.cc
   127.0.0.1 crm.theworklabs.cc
   127.0.0.1 docs.theworklabs.cc
   ```

2. **Create environment files**:
   ```bash
   cp .env.example .env
   cp apps/bigcapital/.env.example apps/bigcapital/.env
   cp apps/corteza/.env.example apps/corteza/.env
   # Edit all files with your configuration values
   ```

3. **Configure certbot** (in `.env`):
   - For local testing: leave `ENABLE_CERTBOT=` empty
   - For production: set `ENABLE_CERTBOT=production`

4. **Create required directories**:
   ```bash
   mkdir -p shared/certbot/www shared/certbot/conf nginx/conf.d
   ```

5. **Start all services**:
   ```bash
   docker-compose up -d
   ```

6. **Access the applications**:
   - BigCapital: `http://accounting.theworklabs.cc`
   - Corteza CRM: `http://crm.theworklabs.cc`
   - Corteza Docs: `http://crm.theworklabs.cc/theworklabs-docs/`

Skip the SSL/HTTPS setup and certbot if you don't need it locally.

## Commands

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f nginx
docker-compose logs -f bigcapital-server

# Stop all
docker-compose down
```

## Adding New Apps

1. Create new folder in `apps/`
2. Add docker-compose.override.yml with service definitions
3. Update nginx.conf with new server block
4. Restart nginx: `docker-compose up -d nginx`
