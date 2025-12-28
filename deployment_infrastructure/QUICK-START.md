# Quick Start - Air-gapped Elastic Stack Deployment

Complete workflow from scratch to running Elasticsearch on Kubernetes.

## Overview

Four simple scripts handle everything:
1. **setup-machine.sh** - Fresh Ubuntu setup (optional)
2. **collect-all.sh** - Collect images and k3s (internet-connected machine)
3. **install-k3s-airgap.sh** - Install Kubernetes (air-gapped machine)
4. **epr.sh** - Deploy registry (air-gapped machine)
5. **deploy.sh** - Deploy Elasticsearch (air-gapped machine)

## Step-by-Step Workflow

### Phase 0: Machine Setup (Optional)

```bash
cd deployment_infrastructure
./setup-machine.sh
# Follow prompts for Docker, Git, etc.
newgrp docker
```

### Phase 1: Collection (Internet-Connected Machine)

```bash
cd deployment_infrastructure
./collect-all.sh

# Enter your images when prompted:
docker.elastic.co/elasticsearch/elasticsearch:9.2.2
docker.elastic.co/kibana/kibana:9.2.2
docker.elastic.co/logstash/logstash:9.2.2
registry:2
[Press Enter]

# When asked about k3s: y

# Output:
# - images/ directory (container .tar files)
# - k3s-files/ directory (k3s components + install script)
```

### Phase 2: Transfer to Air-gapped Machine

```bash
# Copy entire deployment_infrastructure folder
scp -r deployment_infrastructure/ user@airgapped-machine:~/helm-fleet-deployment/
```

### Phase 3: Install Kubernetes (Air-gapped Machine)

```bash
cd deployment_infrastructure
./install-k3s-airgap.sh

# This ONE command does everything:
# ✓ Installs k3s binary and airgap images
# ✓ Configures kubeconfig with permissions
# ✓ Sets up localhost:5000 registry support
# ✓ Verifies installation

# Reload shell
source ~/.bashrc

# Verify
kubectl get nodes
```

### Phase 4: Deploy Registry (Air-gapped Machine)

```bash
cd ../epr_deployment
./epr.sh

# Deploys registry at localhost:5000 with all your images
```

### Phase 5: Deploy Elasticsearch (Air-gapped Machine)

```bash
cd ../helm_charts
./deploy.sh

# Deploys Elasticsearch to Kubernetes
```

### Phase 6: Access Elasticsearch

```bash
# Port-forward
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200

# In another terminal, test
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

## File Locations After Setup

```
~/helm-fleet-deployment/
├── images/                          # Container images
│   └── *.tar
├── k3s-files/                       # k3s installation
│   ├── install-k3s-airgap.sh       # ← RUN THIS
│   ├── k3s
│   ├── k3s-airgap-images-amd64.tar.gz
│   └── install-k3s.sh
├── epr_deployment/
│   └── epr.sh                       # ← RUN THIS
└── helm_charts/
    └── deploy.sh                    # ← RUN THIS
```

## Common Commands

### Kubernetes
```bash
kubectl get nodes                    # Check cluster
kubectl get pods -A                  # All pods
kubectl get namespaces               # Namespaces
```

### Elasticsearch
```bash
kubectl get pods -n elastic          # ES pods
kubectl logs -n elastic -l app=elasticsearch -f
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
```

### Registry
```bash
curl http://localhost:5000/v2/_catalog
docker ps | grep registry
```

### k3s
```bash
sudo systemctl status k3s
sudo systemctl restart k3s
/usr/local/bin/k3s-uninstall.sh     # Uninstall
```

## Troubleshooting

### "Permission denied" on kubectl
```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

### k3s not starting
```bash
sudo systemctl status k3s
sudo journalctl -u k3s -f
```

### Registry not accessible
```bash
docker ps | grep registry
docker logs elastic-registry
```

### Elasticsearch pod not starting
```bash
kubectl describe pod -n elastic elasticsearch-master-0
kubectl logs -n elastic elasticsearch-master-0
```

## Clean Up

```bash
# Remove Elasticsearch
helm uninstall elasticsearch -n elastic
kubectl delete pvc -n elastic -l app=elasticsearch

# Remove registry
cd epr_deployment
./nuke_registry.sh

# Remove k3s
/usr/local/bin/k3s-uninstall.sh
```

## Next Steps

- Scale Elasticsearch: `helm upgrade elasticsearch ./elasticsearch -n elastic --set replicas=3`
- Deploy Kibana: Create similar chart
- Enable security: Configure X-Pack
- Set up backups: Configure snapshots

## Complete Documentation

- [DEPLOYMENT-WORKFLOW.md](DEPLOYMENT-WORKFLOW.md) - Detailed workflow with architecture
- [epr_deployment/README.md](epr_deployment/README.md) - Registry deployment docs
- [epr_deployment/COLLECT-ALL-README.md](epr_deployment/COLLECT-ALL-README.md) - Collection guide
- [helm_charts/README.md](helm_charts/README.md) - Kubernetes deployment docs
