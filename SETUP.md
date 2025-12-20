# Docker Infrastructure Setup Guide

## Initial Setup

### 1. Create Environment Files

```bash
cd docker-infrastructure

# Create BigCapital env file
cp apps/bigcapital/.env.example apps/bigcapital/.env
# Edit and fill in values
nano apps/bigcapital/.env

# Create Corteza env file
cp apps/corteza/.env.example apps/corteza/.env
# Edit and fill in values
nano apps/corteza/.env
```

### 2. Update DNS Records

Add these DNS records (or update /etc/hosts for local testing):

```
accounting.theworklabs.cc    A    <your-ip>
crm.theworklabs.cc          A    <your-ip>
```

For local testing, add to `/etc/hosts`:
```
127.0.0.1 accounting.theworklabs.cc
127.0.0.1 crm.theworklabs.cc
```

Note: Corteza Docs is served as a sub-path at `/theworklabs-docs/` on the same domain.

### 3. Create Required Directories

```bash
mkdir -p shared/certbot/www
mkdir -p shared/certbot/conf
mkdir -p nginx/conf.d
```

### 4. Start Services

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f nginx
docker-compose logs -f bigcapital-server
docker-compose logs -f corteza-server

# Check health
docker-compose ps
```

## Common Commands

### View Logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f nginx
docker-compose logs -f bigcapital-server
docker-compose logs -f corteza-server
```

### Restart Services
```bash
# Restart everything
docker-compose restart

# Restart specific service
docker-compose restart nginx
docker-compose restart bigcapital-server
```

### Stop/Start
```bash
# Stop all
docker-compose down

# Start all
docker-compose up -d

# Remove all (including volumes!)
docker-compose down -v
```

### Update Configuration
```bash
# Edit nginx config
nano nginx/nginx.conf

# Reload nginx without downtime
docker-compose exec nginx nginx -s reload
```

### Execute Commands
```bash
# Access database
docker-compose exec corteza-db psql -U corteza -d corteza

# Access BigCapital MySQL
docker-compose exec bigcapital-mysql mysql -u root -p

# Access Redis
docker-compose exec bigcapital-redis redis-cli
```

## SSL/HTTPS Setup (Optional)

### Using Let's Encrypt with Certbot

```bash
# Create certificate
docker-compose exec certbot certbot certonly \
  -d accounting.theworklabs.cc \
  -d crm.theworklabs.cc \
  -d docs.theworklabs.cc \
  --webroot -w /var/www/certbot

# Update nginx.conf to use SSL (see HTTPS setup below)
```

### Manual SSL Setup

1. Place your certificate at: `shared/certbot/conf/live/[domain]/`
2. Update nginx.conf with SSL configuration
3. Reload nginx

## Adding New Applications

### 1. Update docker-compose.yml

Add your service definition:
```yaml
new-app-server:
  container_name: new-app-server
  image: your-image:latest
  networks:
    - infrastructure_network
  env_file:
    - apps/new-app/.env
  expose:
    - '3000'
```

### 2. Update nginx.conf

Add new server block:
```nginx
upstream new_app {
    server new-app-server:3000;
}

server {
    listen 80;
    server_name new-app.theworklabs.cc;
    
    location / {
        proxy_pass http://new_app;
        # ... proxy headers
    }
}
```

### 3. Create Environment File

```bash
mkdir -p apps/new-app
cp .env.example apps/new-app/.env.example
cp apps/new-app/.env.example apps/new-app/.env
```

### 4. Restart Infrastructure

```bash
docker-compose up -d
docker-compose exec nginx nginx -s reload
```

## Troubleshooting

### Nginx Won't Start
```bash
# Check config syntax
docker-compose run --rm nginx nginx -t

# View error logs
docker-compose logs nginx
```

### Services Can't Reach Each Other
- Verify all services are on same network: `infrastructure_network`
- Check container names match upstream definitions
- Verify expose ports are correct

### Database Connection Issues
- Verify environment variables are set correctly
- Check database containers are running: `docker-compose ps`
- Verify health checks: `docker-compose ps | grep -i healthy`

### Port Already in Use
```bash
# If port 80 is busy
docker-compose down
# Or change port in docker-compose.yml
```

## Performance Tuning

### Connection Pooling
Nginx upstream connections are kept alive with `keepalive 32` for each upstream.

### Gzip Compression
Enabled for text files, JSON, JavaScript - saves bandwidth.

### Client Upload Size
Max body size set to 100M for file uploads.

### Worker Processes
Set to `auto` - adapts to CPU count.

## Security Considerations

1. **Change all default passwords** in .env files
2. **Use strong JWT secrets** (min 32 chars)
3. **Enable HTTPS** using Let's Encrypt or your certificates
4. **Restrict access** if needed using Nginx auth modules
5. **Update images regularly** - set specific versions instead of `latest`
6. **Regular backups** of PostgreSQL and MySQL volumes

## Monitoring

### Docker Stats
```bash
docker stats
```

### Check Disk Space
```bash
docker system df
```

### Volume Usage
```bash
docker volume inspect infrastructure_corteza_db
docker volume inspect infrastructure_bigcapital_mysql
```

## Backup Strategy

### PostgreSQL (Corteza)
```bash
docker-compose exec corteza-db pg_dump -U corteza corteza > backup.sql
```

### MySQL (BigCapital)
```bash
docker-compose exec bigcapital-mysql mysqldump -u root -p --all-databases > backup.sql
```

### MongoDB (BigCapital)
```bash
docker-compose exec bigcapital-mongo mongodump --out /backup/
```

### Restore
```bash
# PostgreSQL
docker-compose exec -T corteza-db psql -U corteza corteza < backup.sql

# MySQL
docker-compose exec -T bigcapital-mysql mysql -u root -p < backup.sql
```
