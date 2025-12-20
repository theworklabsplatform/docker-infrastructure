# Quick Reference

## Directory Structure

```
docker-infrastructure/
├── README.md                 # Overview
├── SETUP.md                  # Detailed setup guide
├── ARCHITECTURE.md           # Technical architecture
├── QUICKREF.md              # This file
├── docker-compose.yml       # Main orchestration file
├── .env.example             # Environment template
│
├── nginx/
│   └── nginx.conf           # Master Nginx config (hostname routing)
│
├── apps/
│   ├── bigcapital/
│   │   └── .env.example     # BigCapital env template
│   └── corteza/
│       └── .env.example     # Corteza env template
│
└── shared/
    ├── certbot/
    │   ├── www/             # ACME challenges
    │   └── conf/            # Certificates
    └── volumes/             # Data backups
```

## Hostnames & Routing

| Hostname | Service | Internal Port | Purpose |
|----------|---------|---------------|---------|
| `accounting.theworklabs.cc` | BigCapital | 3000 (API), 80 (Web) | Accounting & Finance |
| `crm.theworklabs.cc` | Corteza | 80 | CRM System |
| `docs.theworklabs.cc` | Corteza Docs | 80 | Documentation |

## Essential Commands

```bash
# Navigate to infrastructure
cd docker-infrastructure

# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# View logs (follow mode)
docker-compose logs -f nginx

# Check status
docker-compose ps

# Restart specific service
docker-compose restart nginx

# Execute command in container
docker-compose exec corteza-server [command]

# Reload Nginx (without downtime)
docker-compose exec nginx nginx -s reload

# View resource usage
docker stats
```

## Service Details

### BigCapital
- **Frontend**: http://bigcapital-webapp:80 (SPA, React)
- **Backend**: http://bigcapital-server:3000 (Node.js)
- **Database**: MySQL (bigcapital-mysql:3306)
- **Cache**: Redis (bigcapital-redis:6379)
- **Docs**: MongoDB (bigcapital-mongo:27017)
- **PDF**: Gotenberg (bigcapital-gotenberg:9000)

### Corteza
- **Server**: http://corteza-server:80
- **Database**: PostgreSQL (corteza-db:5432)
- **Storage**: /data volume
- **Docs**: corteza-docs:80

## Environment Variables

### Create Config Files
```bash
# BigCapital
cp apps/bigcapital/.env.example apps/bigcapital/.env
nano apps/bigcapital/.env

# Corteza
cp apps/corteza/.env.example apps/corteza/.env
nano apps/corteza/.env
```

### Key Variables

**BigCapital:**
- `DB_USER` / `DB_PASSWORD` - MySQL credentials
- `JWT_SECRET` - Session signing secret (min 32 chars)
- `BASE_URL` - App URL (https://accounting.theworklabs.cc)

**Corteza:**
- `POSTGRES_USER` / `POSTGRES_PASSWORD` - DB credentials
- `CORTEZA_BASE_URL` - App URL (https://crm.theworklabs.cc)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port 80 in use | Change `PUBLIC_PORT` in docker-compose.yml |
| Services can't reach each other | Verify network: `docker network inspect infrastructure_network` |
| Database not initializing | Check logs: `docker-compose logs corteza-db` |
| Nginx config error | Test: `docker-compose exec nginx nginx -t` |
| Container won't start | View error: `docker-compose logs [service-name]` |

## Common Tasks

### Backup Corteza Database
```bash
docker-compose exec -T corteza-db pg_dump \
  -U corteza corteza > corteza_backup.sql
```

### Restore Corteza Database
```bash
docker-compose exec -T corteza-db psql \
  -U corteza corteza < corteza_backup.sql
```

### Backup BigCapital MySQL
```bash
docker-compose exec -T bigcapital-mysql mysqldump \
  -u root -p --all-databases > bigcapital_backup.sql
```

### Clear All Data (⚠️ Destructive)
```bash
docker-compose down -v  # Removes all volumes
```

### Update Service Image
```bash
# Edit docker-compose.yml with new version
docker-compose pull [service-name]
docker-compose up -d [service-name]
```

## Monitoring Checklist

Daily:
- [ ] Check service status: `docker-compose ps`
- [ ] Review logs for errors: `docker-compose logs --tail 100`

Weekly:
- [ ] Check disk usage: `docker system df`
- [ ] Backup databases
- [ ] Review performance: `docker stats`

Monthly:
- [ ] Update Docker images
- [ ] Update base OS packages
- [ ] Check SSL certificate expiry
- [ ] Review security settings

## Adding New Applications

1. Create service in `docker-compose.yml`
2. Add upstream and server block in `nginx.conf`
3. Create `.env` file in `apps/[app-name]/`
4. Restart: `docker-compose up -d && docker-compose exec nginx nginx -s reload`

## SSL/HTTPS Setup

```bash
# Generate certificates with Let's Encrypt
docker-compose exec certbot certbot certonly \
  -d accounting.theworklabs.cc \
  -d crm.theworklabs.cc \
  -d docs.theworklabs.cc \
  --webroot -w /var/www/certbot

# Update nginx.conf with SSL config
# Restart Nginx
docker-compose exec nginx nginx -s reload
```

## Performance Tips

- Use specific image versions (not `latest`)
- Enable gzip compression (already done)
- Set appropriate connection pools
- Monitor and adjust worker processes
- Use Redis for caching
- Regular database maintenance

## Help & Support

1. Check logs: `docker-compose logs [service]`
2. Test Nginx syntax: `docker-compose exec nginx nginx -t`
3. View network: `docker network inspect infrastructure_network`
4. Inspect volume: `docker volume inspect infrastructure_corteza_db`
5. See resource usage: `docker stats`
