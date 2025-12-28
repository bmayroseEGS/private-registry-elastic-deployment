# Quick Start - Air-gapped Elastic Stack Deployment

Complete workflow from scratch to running Elastic Stack (Elasticsearch, Kibana, Logstash) on Kubernetes.

## Overview

Four simple scripts handle everything:
1. **setup-machine.sh** - Fresh Ubuntu setup (optional)
2. **collect-all.sh** - Collect images and k3s (internet-connected machine)
3. **install-k3s-airgap.sh** - Install Kubernetes (air-gapped machine)
4. **epr.sh** - Deploy registry (air-gapped machine)
5. **deploy.sh** - Deploy Elastic Stack (air-gapped machine)

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
scp -r deployment_infrastructure/ user@airgapped-machine:~/private-registry-elastic-deployment/
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

### Phase 5: Deploy Elastic Stack (Air-gapped Machine)

```bash
cd ../helm_charts
./deploy.sh

# Interactively prompts:
# Deploy Elasticsearch? (y/n): y
# Deploy Kibana? (y/n): y
# Deploy Logstash? (y/n): y
```

### Phase 6: Access Services

**Elasticsearch:**
```bash
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

**Kibana (via SSH tunnel from local machine):**
```bash
# On local machine
ssh -i "key.pem" -L 5601:localhost:5601 ubuntu@server-ip

# On remote server
kubectl port-forward -n elastic svc/kibana 5601:5601

# Open browser: http://localhost:5601
```

**Logstash:**
```bash
kubectl port-forward -n elastic svc/logstash 8080:8080
curl -X POST http://localhost:8080 -H 'Content-Type: application/json' -d '{"message":"test"}'
```

## File Locations After Setup

```
~/private-registry-elastic-deployment/
├── deployment_infrastructure/
│   ├── collect-all.sh
│   ├── install-k3s-airgap.sh       # ← RUN THIS
│   ├── k3s-files/                  # k3s installation
│   │   ├── k3s
│   │   └── k3s-airgap-images-amd64.tar
│   └── helm-files/helm
├── epr_deployment/
│   ├── epr.sh                      # ← RUN THIS
│   └── images/                     # Container images
│       └── *.tar
└── helm_charts/
    ├── deploy.sh                   # ← RUN THIS
    ├── elasticsearch/
    ├── kibana/
    └── logstash/
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
# Remove Elastic Stack
helm uninstall elasticsearch kibana logstash -n elastic
kubectl delete pvc -n elastic -l app=elasticsearch

# Remove registry
cd epr_deployment
./nuke_registry.sh

# Remove k3s
cd ../deployment_infrastructure
./uninstall-k3s-complete.sh
```

## Next Steps

- Scale Elasticsearch: `helm upgrade elasticsearch ./elasticsearch -n elastic --set replicas=3`
- Enable security: Configure X-Pack security and TLS
- Set up monitoring: Enable Elastic Stack monitoring
- Configure backups: Set up Elasticsearch snapshots
- Deploy Beats: Add Filebeat, Metricbeat for data collection

## Complete Documentation

- [DEPLOYMENT-WORKFLOW.md](DEPLOYMENT-WORKFLOW.md) - Detailed workflow with architecture
- [epr_deployment/README.md](epr_deployment/README.md) - Registry deployment docs
- [epr_deployment/COLLECT-ALL-README.md](epr_deployment/COLLECT-ALL-README.md) - Collection guide
- [helm_charts/README.md](helm_charts/README.md) - Kubernetes deployment docs
