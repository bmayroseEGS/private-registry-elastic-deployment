# Quick Start Guide - Elasticsearch on Kubernetes

## Prerequisites Checklist

- [ ] Local registry running with Elasticsearch images loaded
- [ ] Kubernetes cluster accessible via kubectl
- [ ] Helm 3.x installed
- [ ] kubectl configured and cluster reachable

## One-Command Deployment

```bash
cd helm_charts
./deploy.sh
```

That's it! The script handles everything.

## Manual Step-by-Step

### 1. Verify Prerequisites

```bash
# Check kubectl
kubectl cluster-info

# Check helm
helm version

# Check registry
curl http://localhost:5000/v2/_catalog
```

### 2. Deploy

```bash
# Install
helm install elasticsearch ./elasticsearch \
  --namespace elastic \
  --create-namespace \
  --wait \
  --timeout 10m

# Check status
kubectl get pods -n elastic
```

### 3. Access Elasticsearch

```bash
# Port-forward
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200

# In another terminal, test
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

## Common Commands

### Check Status
```bash
kubectl get all -n elastic
kubectl get pvc -n elastic
```

### View Logs
```bash
kubectl logs -n elastic -l app=elasticsearch -f
```

### Scale Cluster
```bash
helm upgrade elasticsearch ./elasticsearch \
  --namespace elastic \
  --set replicas=3
```

### Uninstall
```bash
helm uninstall elasticsearch -n elastic
kubectl delete pvc -n elastic -l app=elasticsearch  # Optional: delete data
```

## Configuration Quick Changes

### Change Resource Limits

Edit `elasticsearch/values.yaml`:
```yaml
resources:
  requests:
    memory: "4Gi"
  limits:
    memory: "8Gi"
```

Then upgrade:
```bash
helm upgrade elasticsearch ./elasticsearch -n elastic
```

### Change Storage Size

Edit `elasticsearch/values.yaml`:
```yaml
persistence:
  size: 50Gi  # Change from 30Gi
```

Note: Can't resize existing PVCs easily. Best to set before initial install.

### Use Different Registry

Edit `elasticsearch/values.yaml`:
```yaml
image:
  registry: your-registry:5000
```

## Troubleshooting

### Pod Not Starting
```bash
# Check events
kubectl describe pod -n elastic elasticsearch-master-0

# Check logs
kubectl logs -n elastic elasticsearch-master-0
```

### Can't Access Elasticsearch
```bash
# Check service
kubectl get svc -n elastic

# Check endpoints
kubectl get endpoints -n elastic

# Verify pod is ready
kubectl get pods -n elastic
```

### Image Pull Errors
```bash
# Verify registry is accessible
curl http://localhost:5000/v2/elasticsearch/tags/list

# Check if Docker is configured for insecure registry
# On cluster nodes: cat /etc/docker/daemon.json
```

## Next Steps

- Deploy Kibana for visualization
- Configure persistent snapshots for backups
- Enable X-Pack security for production
- Set up monitoring and alerting
- Configure ingress for external access
