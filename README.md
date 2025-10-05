# Strapi K8s - Quick Reference Card

## 🚀 Deploy in 3 Commands

```bash
# 1. Initialize project
bash 05-init-strapi-project.sh

# 2. Configure Docker for local registry
sudo nano /etc/docker/daemon.json
# Add: {"insecure-registries": ["vts-worker-01-stg-srv:30500"]}
sudo systemctl restart docker

# 3. Deploy everything
bash 06-deploy.sh
```

## 🌐 Access URLs

```
Admin Panel:  http://<HAPROXY_IP>/admin
API Endpoint: http://<HAPROXY_IP>/api
Health Check: http://<HAPROXY_IP>/_health
API Docs:     http://<HAPROXY_IP>/documentation
```

## 📊 Essential Commands

### View Status
```bash
# All resources
kubectl get all -n strapi-staging

# Just pods
kubectl get pods -n strapi-staging

# Storage
kubectl get pvc -n strapi-staging

# Ingress
kubectl get ingress -n strapi-staging
```

### Logs
```bash
# Strapi logs (follow)
kubectl logs -f deployment/strapi -n strapi-staging

# Database logs
kubectl logs -f statefulset/postgres -n strapi-staging

# Specific pod
kubectl logs -f <pod-name> -n strapi-staging

# Previous crashed pod
kubectl logs <pod-name> -n strapi-staging --previous
```

### Shell Access
```bash
# Strapi container
kubectl exec -it deployment/strapi -n strapi-staging -- sh

# Database
kubectl exec -it statefulset/postgres -n strapi-staging -- psql -U strapi
```

### Restart Services
```bash
# Restart Strapi
kubectl rollout restart deployment/strapi -n strapi-staging

# Restart PostgreSQL (careful!)
kubectl rollout restart statefulset/postgres -n strapi-staging
```

### Scale
```bash
# Scale up
kubectl scale deployment strapi -n strapi-staging --replicas=3

# Scale down
kubectl scale deployment strapi -n strapi-staging --replicas=1
```

### Update Image
```bash
# After building new image
docker build -t vts-worker-01-stg-srv:30500/strapi:latest strapi-demo/
docker push vts-worker-01-stg-srv:30500/strapi:latest
kubectl rollout restart deployment/strapi -n strapi-staging
```

## 🔧 Database Operations

### Connect to Database
```bash
kubectl exec -it statefulset/postgres -n strapi-staging -- psql -U strapi
```

### Common SQL Commands
```sql
-- List databases
\l

-- Connect to strapi database
\c strapi

-- List tables
\dt

-- View table structure
\d <table_name>

-- Count records
SELECT COUNT(*) FROM <table_name>;

-- Exit
\q
```

### Backup Database
```bash
# Backup to file
kubectl exec statefulset/postgres -n strapi-staging -- \
  pg_dump -U strapi strapi > backup-$(date +%Y%m%d-%H%M%S).sql

# Restore from file
cat backup.sql | kubectl exec -i statefulset/postgres -n strapi-staging -- \
  psql -U strapi strapi
```

## 🐛 Quick Troubleshooting

### Pod Won't Start
```bash
kubectl describe pod <pod-name> -n strapi-staging
kubectl logs <pod-name> -n strapi-staging
kubectl get events -n strapi-staging --sort-by='.lastTimestamp'
```

### Can't Access Strapi
```bash
# Check ingress
kubectl describe ingress strapi-ingress -n strapi-staging

# Check service
kubectl get svc strapi-service -n strapi-staging

# Port forward (temporary access)
kubectl port-forward svc/strapi-service 1337:1337 -n strapi-staging
# Then access: http://localhost:1337/admin
```

### Database Connection Issues
```bash
# Check if database pod is running
kubectl get pods -l app=postgres -n strapi-staging

# Test connection from Strapi pod
kubectl exec -it deployment/strapi -n strapi-staging -- \
  nc -zv postgres-service 5432
```

### Image Pull Errors
```bash
# Check registry
kubectl get pods -n container-registry

# Verify image exists
curl http://vts-worker-01-stg-srv:30500/v2/strapi/tags/list

# Verify Docker daemon config
cat /etc/docker/daemon.json
sudo systemctl status docker
```

## 📈 Performance & Monitoring

### Resource Usage
```bash
# Pod resources
kubectl top pods -n strapi-staging

# Node resources
kubectl top nodes

# Detailed pod info
kubectl describe pod <pod-name> -n strapi-staging | grep -A 5 "Requests"
```

### Watch Resources
```bash
# Watch pods
watch kubectl get pods -n strapi-staging

# Watch events
kubectl get events -n strapi-staging -w
```

## 🔐 Secrets Management

### View Secrets (Encoded)
```bash
kubectl get secrets -n strapi-staging
kubectl describe secret strapi-secret -n strapi-staging
```

### Edit Secrets
```bash
# Edit Strapi secrets
kubectl edit secret strapi-secret -n strapi-staging

# Edit database secrets
kubectl edit secret postgres-secret -n strapi-staging
```

### Generate New Secrets
```bash
# Generate random secret
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

# Or use openssl
openssl rand -base64 32
```

## 📦 Registry Operations

### List Images
```bash
curl http://vts-worker-01-stg-srv:30500/v2/_catalog
```

### List Tags
```bash
curl http://vts-worker-01-stg-srv:30500/v2/strapi/tags/list
```

### Registry Logs
```bash
kubectl logs -f -l app=docker-registry -n container-registry
```

## 🧹 Cleanup Commands

### Remove Strapi (Keep Database)
```bash
kubectl delete deployment strapi -n strapi-staging
kubectl delete svc strapi-service -n strapi-staging
kubectl delete ingress strapi-ingress -n strapi-staging
```

### Complete Cleanup
```bash
# Delete everything in strapi-staging
kubectl delete namespace strapi-staging

# Delete registry (optional)
kubectl delete namespace container-registry
```

### Clean Docker Images
```bash
# Remove local images
docker rmi vts-worker-01-stg-srv:30500/strapi:latest

# Clean build cache
docker system prune -a
```

## 📝 File Locations

```
/opt/app/                          # Strapi root (in container)
/opt/app/public/uploads/           # Uploaded media
/opt/app/config/                   # Configuration files
/var/lib/postgresql/data/pgdata/   # PostgreSQL data (in container)
```

## 🎯 Common Tasks

### Add New Content Type
1. Access admin panel
2. Content-Type Builder → Create new content type
3. Add fields
4. Save
5. (Deployment auto-restarts if needed)

### Update Strapi Code
```bash
cd strapi-demo
# Make your changes
docker build -t vts-worker-01-stg-srv:30500/strapi:latest .
docker push vts-worker-01-stg-srv:30500/strapi:latest
kubectl rollout restart deployment/strapi -n strapi-staging
```

### Check Deployment Status
```bash
kubectl rollout status deployment/strapi -n strapi-staging
```

### View Deployment History
```bash
kubectl rollout history deployment/strapi -n strapi-staging
```

### Rollback Deployment
```bash
# Rollback to previous version
kubectl rollout undo deployment/strapi -n strapi-staging

# Rollback to specific revision
kubectl rollout undo deployment/strapi -n strapi-staging --to-revision=2
```

## 🆘 Emergency Procedures

### Database is Down
```bash
# Check status
kubectl get pods -l app=postgres -n strapi-staging

# View logs
kubectl logs -f statefulset/postgres -n strapi-staging

# Restart (last resort)
kubectl delete pod postgres-0 -n strapi-staging
```

### Strapi Won't Respond
```bash
# Check health
curl http://<HAPROXY_IP>/_health

# View logs
kubectl logs -f deployment/strapi -n strapi-staging

# Force restart
kubectl rollout restart deployment/strapi -n strapi-staging
```

### Out of Storage
```bash
# Check PVC usage
kubectl exec -it deployment/strapi -n strapi-staging -- df -h

# Expand PVC (if supported)
kubectl patch pvc strapi-uploads-pvc -n strapi-staging \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

---
