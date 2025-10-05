# Strapi K8s Deployment Guide - Complete Setup

This guide will help you deploy a production-ready Strapi v5 instance on your Kubernetes cluster.

## 🎯 What You're Building

A complete, enterprise-grade Strapi CMS deployment with:
- ✅ Stateful PostgreSQL with Ceph storage
- ✅ Scalable Strapi deployment (2 replicas)
- ✅ Local container registry
- ✅ Proper secrets management
- ✅ Health checks and resource limits
- ✅ Ingress for direct IP access

## 📋 Prerequisites

### On Your Control/Management Machine:
- kubectl configured and connected to your cluster
- Docker installed and running
- Node.js 18+ and npm (for building Strapi)
- Internet access (for initial dependencies)

### Verify Your Cluster:
```bash
kubectl get nodes
kubectl get storageclass  # Should show ceph-rbd
kubectl get ingressclass  # Should show nginx
```

## 🚀 Quick Start (30 Minutes)

### Step 1: Download All Files

Save all the provided artifacts in a single directory:
```
strapi-k8s/
├── 00-DEPLOYMENT-GUIDE.md          # This file
├── 01-local-registry.yaml          # Container registry
├── 02-postgresql.yaml              # Database
├── 03-Dockerfile                   # Strapi container
├── 04-strapi-deployment.yaml       # Strapi app
├── 05-init-strapi-project.sh       # Project initialization
└── 06-deploy.sh                    # Automated deployment
```

### Step 2: Initialize Strapi Project

```bash
# Make scripts executable
chmod +x 05-init-strapi-project.sh 06-deploy.sh

# Create a fresh Strapi v5 project
bash 05-init-strapi-project.sh
```

This creates a `strapi-demo` directory with:
- Strapi v5 application
- PostgreSQL configuration
- Production-ready configs
- TypeScript support

### Step 3: Configure Docker for Insecure Registry

Edit `/etc/docker/daemon.json` (create if doesn't exist):
```json
{
  "insecure-registries": ["vts-worker-01-stg-srv:30500"]
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

**Why?** The local registry doesn't have SSL, so Docker needs to trust it.

### Step 4: Deploy Everything

```bash
bash 06-deploy.sh
```

This script will:
1. ✅ Deploy local container registry
2. ✅ Deploy PostgreSQL with persistent storage
3. ✅ Build Strapi Docker image
4. ✅ Push image to local registry
5. ✅ Deploy Strapi to K8s
6. ✅ Display access information

## 🔍 Manual Step-by-Step (If You Prefer)

### 1. Deploy Container Registry

```bash
kubectl apply -f 01-local-registry.yaml

# Wait for registry
kubectl wait --for=condition=ready pod -l app=docker-registry -n container-registry --timeout=120s

# Verify registry is accessible
curl http://vts-worker-01-stg-srv:30500/v2/_catalog
```

### 2. Create Strapi Namespace & Deploy PostgreSQL

```bash
# Create namespace
kubectl create namespace strapi-staging

# Deploy PostgreSQL
kubectl apply -f 02-postgresql.yaml

# Wait for PostgreSQL
kubectl wait --for=condition=ready pod -l app=postgres -n strapi-staging --timeout=300s

# Verify database
kubectl exec -it statefulset/postgres -n strapi-staging -- psql -U strapi -c "\l"
```

### 3. Build and Push Strapi Image

```bash
# Copy Dockerfile to project
cp 03-Dockerfile strapi-demo/Dockerfile

# Build image
cd strapi-demo
docker build -t vts-worker-01-stg-srv:30500/strapi:latest .

# Push to registry
docker push vts-worker-01-stg-srv:30500/strapi:latest

cd ..
```

### 4. Deploy Strapi Application

```bash
kubectl apply -f 04-strapi-deployment.yaml

# Watch deployment
kubectl get pods -n strapi-staging -w

# Check logs
kubectl logs -f deployment/strapi -n strapi-staging
```

## 🌐 Accessing Strapi

### Find Your HAProxy IP:
```bash
# On HAProxy server
hostname -I | awk '{print $1}'
```

### Access Strapi Admin:
```
http://<HAPROXY_IP>/admin
```

On first access, you'll be prompted to create an admin account.

## 🔧 Useful Commands

### View All Resources:
```bash
kubectl get all -n strapi-staging
kubectl get pvc -n strapi-staging
kubectl get ingress -n strapi-staging
```

### Check Logs:
```bash
# Strapi logs
kubectl logs -f deployment/strapi -n strapi-staging

# PostgreSQL logs
kubectl logs -f statefulset/postgres -n strapi-staging

# Follow specific pod
kubectl logs -f <pod-name> -n strapi-staging
```

### Shell Access:
```bash
# Strapi container
kubectl exec -it deployment/strapi -n strapi-staging -- sh

# PostgreSQL container
kubectl exec -it statefulset/postgres -n strapi-staging -- psql -U strapi
```

### Scale Strapi:
```bash
# Scale to 3 replicas
kubectl scale deployment strapi -n strapi-staging --replicas=3

# Scale to 1 replica
kubectl scale deployment strapi -n strapi-staging --replicas=1
```

### Restart Strapi:
```bash
kubectl rollout restart deployment/strapi -n strapi-staging
```

### Update Strapi Image:
```bash
# After building new image
kubectl set image deployment/strapi strapi=vts-worker-01-stg-srv:30500/strapi:latest -n strapi-staging

# Or force pull
kubectl rollout restart deployment/strapi -n strapi-staging
```

## 🔐 Security Notes

### Change Default Passwords:

1. **PostgreSQL Password:**
```bash
kubectl edit secret postgres-secret -n strapi-staging
# Change POSTGRES_PASSWORD (base64 encoded)
```

2. **Strapi Secrets:**
```bash
kubectl edit secret strapi-secret -n strapi-staging
# Generate new secrets with: node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

3. **Recommended: Use External Secrets Operator or Sealed Secrets in production**

## 📊 Monitoring & Health Checks

### Check Pod Health:
```bash
kubectl describe pod <pod-name> -n strapi-staging
```

### View Events:
```bash
kubectl get events -n strapi-staging --sort-by='.lastTimestamp'
```

### Resource Usage:
```bash
kubectl top pods -n strapi-staging
kubectl top nodes
```

### Health Endpoints:
- Strapi: `http://<HAPROXY_IP>/_health`
- Admin: `http://<HAPROXY_IP>/admin`
- API: `http://<HAPROXY_IP>/api`

## 🐛 Troubleshooting

### Strapi Pod Won't Start

```bash
# Check pod status
kubectl describe pod -l app=strapi -n strapi-staging

# Common issues:
# 1. Database not ready - check postgres logs
# 2. Image pull error - verify registry accessibility
# 3. Resource constraints - check node resources
```

### Database Connection Issues

```bash
# Test database connectivity
kubectl run -it --rm debug --image=postgres:16-alpine --restart=Never -n strapi-staging -- \
  psql -h postgres-service -U strapi -d strapi

# Check database service
kubectl get svc postgres-service -n strapi-staging
```

### Image Build Failures

```bash
# Build with verbose output
docker build --no-cache --progress=plain -t vts-worker-01-stg-srv:30500/strapi:latest .

# Check Docker daemon logs
sudo journalctl -u docker -f
```

### Registry Issues

```bash
# Verify registry pod
kubectl get pods -n container-registry

# Check registry logs
kubectl logs -l app=docker-registry -n container-registry

# Test registry access
curl http://vts-worker-01-stg-srv:30500/v2/_catalog
```

### Ingress Not Working

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress resource
kubectl describe ingress strapi-ingress -n strapi-staging

# Verify nginx ingress logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

## 📈 Next Steps & Production Readiness

### For Your Development Team:

1. **Show them the running instance**
   - Admin panel: `http://<HAPROXY_IP>/admin`
   - API docs: `http://<HAPROXY_IP>/documentation`

2. **Gather requirements:**
   - What content types do they need?
   - What plugins are they using?
   - What integrations exist?
   - What's their workflow?

3. **Plan migration:**
   - Export their local schemas
   - Document custom code
   - List dependencies
   - Plan data migration strategy

### For Production Deployment:

- [ ] Set up proper DNS and SSL certificates
- [ ] Implement proper secrets management (Vault, Sealed Secrets)
- [ ] Configure backup strategy for PostgreSQL
- [ ] Set up monitoring (Prometheus/Grafana)
- [ ] Configure logging aggregation (ELK/Loki)
- [ ] Implement CI/CD pipeline (Azure DevOps)
- [ ] Set up staging → production promotion
- [ ] Configure resource quotas and limits
- [ ] Implement network policies
- [ ] Set up disaster recovery plan

### DevSecOps Checklist:

- [ ] Git repository with proper branching strategy
- [ ] Automated testing in pipeline
- [ ] Container vulnerability scanning
- [ ] RBAC for K8s access
- [ ] Audit logging enabled
- [ ] Regular backup testing
- [ ] Documentation for team
- [ ] Runbooks for common issues
- [ ] On-call rotation setup

## 📝 Architecture Diagram

```
Internet/Internal Network
         |
         v
    [HAProxy LB]
         |
    Port 80 → NodePort 30080
         |
         v
  [Nginx Ingress Controller]
         |
    [Ingress Resource]
         |
    +----+----+
    |         |
    v         v
[Strapi]  [Strapi]     ← 2 replicas, stateless
(Pod 1)   (Pod 2)      ← Shared uploads via PVC
    |         |
    +----+----+
         |
         v
  [Strapi Service]
         |
         v
  [PostgreSQL StatefulSet]
         |
         v
   [Ceph-RBD PVC]        ← 20Gi persistent storage
```

## 🎓 Learning Resources

- Strapi Documentation: https://docs.strapi.io
- Kubernetes Best Practices: https://kubernetes.io/docs/concepts/
- Rook-Ceph: https://rook.io/docs/
- Nginx Ingress: https://kubernetes.github.io/ingress-nginx/

## 💡 Tips for Success

1. **Start Simple:** Get basic setup working first, then add complexity
2. **Document Everything:** Your team will thank you
3. **Test Backups:** Don't wait for disaster to test restores
4. **Monitor Proactively:** Set up alerts before issues occur
5. **Iterate:** Improve based on team feedback

## 🆘 Support

If you encounter issues:
1. Check the troubleshooting section above
2. Review pod logs: `kubectl logs -f <pod-name> -n strapi-staging`
3. Check K8s events: `kubectl get events -n strapi-staging`
4. Verify network connectivity between pods

---

**Good luck with your deployment! You're building a solid foundation for your team's DevSecOps culture. 🚀**
