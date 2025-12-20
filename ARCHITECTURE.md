# Infrastructure Architecture

## Network Topology

```
Internet (Port 80/443)
        ↓
   ┌────────────┐
   │ Nginx      │ (infrastructure-nginx)
   │ Proxy      │
   └────┬───────┘
        ├─────────────────────┬──────────────────────┐
        │                     │                      │
   ┌────▼────────┐    ┌──────▼──────┐    ┌─────────▼─────┐
   │ BigCapital  │    │   Corteza   │    │  Corteza Docs │
   │ Accounting  │    │    CRM      │    │                │
   └─────────────┘    └─────────────┘    └────────────────┘
        │                     │
        ├──────┬──────────────┴────┐
        │      │                   │
   ┌────▼──────▼──┐    ┌──────────▼────┐
   │              │    │                │
   │ BigCapital   │    │   Corteza DB   │
   │ Services     │    │  (PostgreSQL)  │
   │              │    │                │
   │ ├─Server     └────────────────────┘
   │ ├─Webapp
   │ ├─MySQL
   │ ├─MongoDB
   │ ├─Redis
   │ └─Gotenberg
   └─────────────┘
```

## Request Flow

### BigCapital (accounting.theworklabs.cc)

```
User Request
    ↓
Nginx (hostname: accounting.theworklabs.cc)
    ├─ /api/* → bigcapital-server:3000 (API)
    ├─ /socket.io/* → bigcapital-server:3000 (WebSockets)
    └─ /* → bigcapital-webapp:80 (React SPA)
        ↓
    Services
    ├─ MySQL (bigcapital-mysql:3306)
    ├─ MongoDB (bigcapital-mongo:27017)
    ├─ Redis (bigcapital-redis:6379)
    └─ Gotenberg (bigcapital-gotenberg:9000)
```

### Corteza (crm.theworklabs.cc)

```
User Request
    ↓
Nginx (hostname: crm.theworklabs.cc)
    ├─ /* → corteza-server:80
        ↓
        PostgreSQL (corteza-db:5432)
```

### Corteza Docs (docs.theworklabs.cc)

```
User Request
    ↓
Nginx (hostname: docs.theworklabs.cc)
    └─ /* → corteza-docs:80
```

## Service Communication

All services communicate through the `infrastructure_network` Docker bridge network:

- Container names resolve internally (e.g., `bigcapital-server` = `http://bigcapital-server:3000`)
- No external port exposure except Nginx (port 80/443)
- Services are isolated in their own network

## Data Persistence

### Named Volumes

```
Infrastructure Volumes:
├─ infrastructure_bigcapital_mysql → BigCapital relational data
├─ infrastructure_bigcapital_mongo → BigCapital document store
├─ infrastructure_bigcapital_redis → BigCapital cache/sessions
├─ infrastructure_corteza_db → Corteza relational data
└─ infrastructure_corteza_data → Corteza file storage
```

### Backup Strategy

Each volume can be backed up independently:

```bash
# Backup MongoDB
docker run --rm -v infrastructure_bigcapital_mongo:/mongo \
  mongo:latest mongodump --out /backup/

# Backup MySQL
docker run --rm -v infrastructure_bigcapital_mysql:/mysql \
  mysql:latest mysqldump --all-databases
```

## Scaling Considerations

### Current Setup (Single Machine)

✓ All services on one machine
✓ Simple management
✓ Local development/testing
✗ Single point of failure
✗ Limited by machine resources

### Scaling to Multiple Machines

1. **Load Balancing**: Add HAProxy or another load balancer in front of Nginx
2. **Database Separation**: Move databases to separate machines
3. **Microservices**: Split BigCapital into multiple backend services
4. **Kubernetes**: Migrate to K8s for better orchestration

### To Add Multiple BigCapital Backends

```yaml
# In docker-compose.yml
bigcapital-server-2:
  # ... copy of bigcapital-server with different ports

# In nginx.conf upstream
upstream bigcapital_backend {
    server bigcapital-server:3000;
    server bigcapital-server-2:3000;
}
```

## Security Architecture

### Network Isolation
- Services only expose ports on internal bridge network
- Only Nginx exposes ports to the host
- No direct external access to databases

### DNS Resolution
- Services use Docker's internal DNS (127.0.0.11:53)
- Container names resolve to IPs automatically

### SSL/TLS
```
Client (HTTPS)
    ↓
Nginx (SSL termination)
    ↓
Backend Services (HTTP internal)
```

## Monitoring & Logging

### Log Aggregation
```bash
# Centralized logging (all containers)
docker-compose logs -f

# Per-service logging
docker-compose logs -f nginx
docker-compose logs -f bigcapital-server
```

### Health Checks
Each service has health check endpoints:
- Nginx: `GET /health`
- BigCapital: `GET /api/health`
- Corteza: `GET /`

Docker monitors these and restarts unhealthy containers.

### Metrics
Monitor using Docker's built-in tools:
```bash
docker stats              # Live resource usage
docker system df          # Disk usage
docker volume ls          # Volume list
```

## Database Schema Relationships

### BigCapital

```
MySQL (System DB)
├─ tenants (multi-tenant support)
├─ system_users
├─ plans
└─ subscriptions

MySQL (Tenant DBs) - One per organization
├─ accounts
├─ items
├─ contacts
├─ invoices
├─ bills
├─ expenses
└─ transactions

MongoDB (Audit logs & sessions)

Redis (Cache & sessions)
```

### Corteza

```
PostgreSQL
├─ rbac (RBAC rules)
├─ compose (Custom data models)
├─ system (System configuration)
├─ users
├─ roles
└─ permissions
```

## Disaster Recovery

### RTO (Recovery Time Objective): ~10 minutes
### RPO (Recovery Point Objective): Last backup

**Procedure:**
1. Restore databases from backup
2. Restart containers
3. Verify data consistency
4. Resume operations

## Future Enhancements

1. **SSL/HTTPS**: Let's Encrypt integration
2. **Monitoring**: Prometheus + Grafana
3. **Log Centralization**: ELK Stack
4. **Container Registry**: Private Docker registry
5. **CI/CD**: Automated deployments
6. **Database Replication**: High availability
7. **CDN**: CloudFlare for static assets
