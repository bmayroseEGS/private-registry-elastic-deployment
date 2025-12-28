# Air-gapped Elastic Stack Deployment

Complete solution for deploying Elastic Stack (Elasticsearch, Kibana, Logstash) in air-gapped Kubernetes environments using a local container registry.

## Overview

This project provides a complete workflow for deploying Elastic Stack in environments without internet access:

1. **Image Collection** - Collect and package container images on internet-connected machine
2. **Registry Deployment** - Deploy local Docker registry and load images on air-gapped machine
3. **Kubernetes Deployment** - Deploy Elasticsearch to Kubernetes using Helm charts

## Quick Start

### Phase 1: Collect Images (Internet-connected machine)
```bash
cd epr_deployment
./collect-images.sh
# Follow prompts to enter image URLs
# Images saved to images/*.tar
```

### Phase 2: Transfer & Deploy Registry (Air-gapped machine)
```bash
# Transfer images/ directory to air-gapped machine
cd epr_deployment
./epr.sh
# Registry deployed at localhost:5000
```

### Phase 3: Deploy to Kubernetes
```bash
cd helm_charts
./deploy.sh
# Elasticsearch deployed to Kubernetes
```

## Project Structure

```
├── epr_deployment/           # Container registry deployment
│   ├── collect-images.sh    # Collect images from internet
│   ├── epr.sh               # Deploy local registry
│   ├── nuke_registry.sh     # Cleanup script
│   └── images/              # Collected .tar files (gitignored)
│
├── helm_charts/             # Kubernetes Helm charts
│   ├── deploy.sh            # Kubernetes deployment script
│   └── elasticsearch/       # Elasticsearch Helm chart
│       ├── Chart.yaml
│       ├── values.yaml      # Configuration
│       └── templates/       # K8s manifests
│
└── DEPLOYMENT-WORKFLOW.md   # Complete workflow guide
```

## Documentation

- **[DEPLOYMENT-WORKFLOW.md](DEPLOYMENT-WORKFLOW.md)** - Complete end-to-end workflow with architecture diagrams
- **[epr_deployment/README.md](epr_deployment/README.md)** - Registry deployment documentation
- **[helm_charts/README.md](helm_charts/README.md)** - Kubernetes/Helm deployment documentation

## Quick References

- **[epr_deployment/QUICKSTART.md](epr_deployment/QUICKSTART.md)** - Registry quick reference
- **[helm_charts/QUICKSTART.md](helm_charts/QUICKSTART.md)** - Kubernetes quick reference

## Features

### Registry Deployment (epr_deployment)
- Interactive image URL collection
- Automatic Docker insecure registry configuration
- Image loading and tagging for local registry
- Registry health verification
- Complete cleanup script

### Kubernetes Deployment (helm_charts)
- Production-ready Helm chart for Elasticsearch
- Configurable resources, replicas, and persistence
- StatefulSet with proper health checks
- Persistent storage with PVCs
- Single-node and multi-node cluster support
- Automated deployment script

## Prerequisites

**Internet-connected Machine:**
- Docker installed
- Internet access

**Air-gapped Machine:**
- Docker installed and running
- Kubernetes cluster (minikube, k3s, or production cluster)
- kubectl configured
- Helm 3.x installed

## Architecture

```
Internet Machine          Air-gapped Machine              Kubernetes Cluster
┌──────────────┐         ┌──────────────────┐           ┌─────────────────┐
│              │         │                  │           │                 │
│ collect-     │  .tar   │  epr.sh          │  images   │  deploy.sh      │
│ images.sh    │-------->│  ↓               │---------->│  ↓              │
│              │  files  │  Local Registry  │           │  Elasticsearch  │
│              │         │  localhost:5000  │           │  Pods           │
└──────────────┘         └──────────────────┘           └─────────────────┘
```

## Example Workflow

```bash
# 1. On internet-connected machine
cd epr_deployment
./collect-images.sh
# Enter: docker.elastic.co/elasticsearch/elasticsearch:9.2.2
# Enter: docker.elastic.co/kibana/kibana:9.2.2
# Enter: registry:2
# Press Enter to finish

# 2. Transfer images/ to air-gapped machine
scp -r images/ user@airgapped:/path/to/epr_deployment/

# 3. On air-gapped machine - deploy registry
cd epr_deployment
./epr.sh
# Registry starts at localhost:5000 with all images loaded

# 4. Deploy to Kubernetes
cd ../helm_charts
./deploy.sh
# Elasticsearch deployed to 'elastic' namespace

# 5. Access Elasticsearch
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
curl http://localhost:9200
```

## Configuration

### Registry Configuration
Edit images in [epr_deployment/sample-image-list.txt](epr_deployment/sample-image-list.txt)

### Kubernetes Configuration
Edit [helm_charts/elasticsearch/values.yaml](helm_charts/elasticsearch/values.yaml):
```yaml
image:
  registry: localhost:5000  # Local registry
  tag: "9.2.2"             # Version

replicas: 1                # Start with 1, scale to 3+

resources:
  limits:
    memory: "4Gi"          # Adjust as needed
```

## Scaling to Production

```bash
# Scale to 3 nodes
helm upgrade elasticsearch ./elasticsearch \
  --namespace elastic \
  --set replicas=3

# Increase resources
helm upgrade elasticsearch ./elasticsearch \
  --namespace elastic \
  --set resources.limits.memory=8Gi \
  --set esJavaOpts="-Xmx4g -Xms4g"
```

## Troubleshooting

### Registry Issues
```bash
# Check registry
curl http://localhost:5000/v2/_catalog

# View registry logs
docker logs elastic-registry

# Restart registry
docker restart elastic-registry
```

### Kubernetes Issues
```bash
# Check pods
kubectl get pods -n elastic

# View logs
kubectl logs -n elastic -l app=elasticsearch -f

# Check events
kubectl get events -n elastic
```

## Cleanup

```bash
# Remove Kubernetes deployment
helm uninstall elasticsearch -n elastic
kubectl delete pvc -n elastic -l app=elasticsearch

# Remove registry
cd epr_deployment
./nuke_registry.sh
```

## Next Steps

1. Deploy Kibana using similar Helm chart
2. Deploy Logstash for data processing
3. Enable X-Pack security for production
4. Configure ingress for external access
5. Set up monitoring and alerting
6. Configure snapshot backups

## Support

For detailed information, see:
- [Complete Workflow Guide](DEPLOYMENT-WORKFLOW.md)
- [Registry Documentation](epr_deployment/README.md)
- [Helm Chart Documentation](helm_charts/README.md)

## License

This is an internal deployment tool. Elastic Stack components are subject to Elastic License.
