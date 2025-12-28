# Complete Air-gapped Elastic Stack Deployment Workflow

This guide shows the complete workflow from collecting images to deploying Elasticsearch on Kubernetes in an air-gapped environment.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT WORKFLOW                               │
└─────────────────────────────────────────────────────────────────────────┘

PHASE 1: Image Collection (Internet-connected Machine)
┌─────────────────────────────────────────────────────────────┐
│  Internet-connected Machine                                 │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./collect-images.sh                    │          │
│  │                                               │          │
│  │  → Prompts for image URLs                    │          │
│  │  → Pulls from Docker Hub / Elastic registry  │          │
│  │  → Saves as .tar files                       │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Output: images/*.tar                        │          │
│  │  - elasticsearch-9.2.2.tar                   │          │
│  │  - kibana-9.2.2.tar                          │          │
│  │  - logstash-9.2.2.tar                        │          │
│  │  - registry-2.tar                            │          │
│  └───────────────┬──────────────────────────────┘          │
└──────────────────┼──────────────────────────────────────────┘
                   │
                   │ Transfer via USB/SCP
                   ▼

PHASE 2: Registry Deployment (Air-gapped Machine)
┌─────────────────────────────────────────────────────────────┐
│  Air-gapped Machine                                         │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./epr.sh                               │          │
│  │                                               │          │
│  │  → Configures Docker insecure registry       │          │
│  │  → Starts Docker Registry container          │          │
│  │  → Loads .tar files into Docker              │          │
│  │  → Tags and pushes to localhost:5000         │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Docker Registry: localhost:5000             │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  elasticsearch:9.2.2                   │  │          │
│  │  │  kibana:9.2.2                          │  │          │
│  │  │  logstash:9.2.2                        │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  └───────────────┬──────────────────────────────┘          │
└──────────────────┼──────────────────────────────────────────┘
                   │
                   │ Images available at localhost:5000
                   ▼

PHASE 3: Kubernetes Deployment
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (same or different air-gapped machine)  │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./deploy.sh                            │          │
│  │                                               │          │
│  │  → Checks prerequisites                      │          │
│  │  → Creates namespace                         │          │
│  │  → Deploys Helm chart                        │          │
│  │  → Waits for pods to be ready                │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Namespace: elastic                          │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  StatefulSet: elasticsearch-master     │  │          │
│  │  │  ┌──────────────────────────────────┐  │  │          │
│  │  │  │  Pod: elasticsearch-master-0     │  │  │          │
│  │  │  │  Image: localhost:5000/          │  │  │          │
│  │  │  │         elasticsearch:9.2.2      │  │  │          │
│  │  │  │  Ports: 9200, 9300               │  │  │          │
│  │  │  │  Volume: 30Gi PVC                │  │  │          │
│  │  │  └──────────────────────────────────┘  │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  Services:                             │  │          │
│  │  │  - elasticsearch-master (ClusterIP)    │  │          │
│  │  │  - elasticsearch-master-headless       │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

## Step-by-Step Workflow

### Prerequisites

**Internet-connected Machine:**
- Docker installed
- Internet access to Docker Hub and Elastic registries

**Air-gapped Machine:**
- Docker installed and running
- Kubernetes cluster (minikube, k3s, or full cluster)
- kubectl configured
- Helm 3.x installed

### Phase 1: Collect Images (Internet-connected)

1. **Clone the repository**
   ```bash
   git clone <repo-url>
   cd helm-fleet-deployment/epr_deployment
   ```

2. **Run the collection script**
   ```bash
   ./collect-images.sh
   ```

3. **Enter image URLs when prompted**
   ```
   Image URL [1]: docker.elastic.co/elasticsearch/elasticsearch:9.2.2
   Image URL [2]: docker.elastic.co/kibana/kibana:9.2.2
   Image URL [3]: docker.elastic.co/logstash/logstash:9.2.2
   Image URL [4]: registry:2
   Image URL [5]: [press Enter to finish]
   ```

4. **Wait for completion**
   - Script will pull each image
   - Save as .tar files in `images/` directory
   - This may take 10-30 minutes depending on image sizes

5. **Verify output**
   ```bash
   ls -lh images/
   # Should show .tar files for each image
   ```

### Phase 2: Transfer Images

**Option A: USB Drive**
```bash
# Copy entire images directory to USB
cp -r images/ /media/usb/

# On air-gapped machine, copy back
cp -r /media/usb/images/ ~/helm-fleet-deployment/epr_deployment/
```

**Option B: SCP (if limited network access)**
```bash
scp -r images/ user@airgapped-machine:~/helm-fleet-deployment/epr_deployment/
```

### Phase 3: Deploy Registry (Air-gapped)

1. **Navigate to deployment directory**
   ```bash
   cd ~/helm-fleet-deployment/epr_deployment
   ```

2. **Verify images are present**
   ```bash
   ls -lh images/
   ```

3. **Run the EPR deployment script**
   ```bash
   ./epr.sh
   ```

4. **Follow prompts**
   - Configure Docker insecure registry (auto or manual)
   - Wait for registry to start
   - Wait for images to load and push

5. **Verify registry**
   ```bash
   curl http://localhost:5000/v2/_catalog
   # Should show: {"repositories":["elasticsearch","kibana","logstash"]}
   ```

### Phase 4: Deploy to Kubernetes

1. **Navigate to helm charts**
   ```bash
   cd ../helm_charts
   ```

2. **Review configuration** (optional)
   ```bash
   cat elasticsearch/values.yaml
   # Check registry URL, resources, replicas, etc.
   ```

3. **Deploy using script**
   ```bash
   ./deploy.sh
   ```

4. **Or deploy manually**
   ```bash
   helm install elasticsearch ./elasticsearch \
     --namespace elastic \
     --create-namespace \
     --wait \
     --timeout 10m
   ```

5. **Verify deployment**
   ```bash
   kubectl get pods -n elastic
   kubectl get svc -n elastic
   kubectl get pvc -n elastic
   ```

### Phase 5: Access Elasticsearch

1. **Port-forward to access**
   ```bash
   kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
   ```

2. **Test connection** (in another terminal)
   ```bash
   curl http://localhost:9200
   curl http://localhost:9200/_cluster/health?pretty
   ```

3. **Expected output**
   ```json
   {
     "cluster_name" : "elasticsearch",
     "status" : "green",
     "number_of_nodes" : 1,
     "number_of_data_nodes" : 1
   }
   ```

## Directory Structure

```
helm-fleet-deployment/
├── epr_deployment/                 # Registry deployment
│   ├── collect-images.sh          # Phase 1: Collect images
│   ├── epr.sh                     # Phase 3: Deploy registry
│   ├── nuke_registry.sh           # Cleanup script
│   ├── images/                    # Image .tar files (gitignored)
│   ├── README.md                  # EPR documentation
│   ├── QUICKSTART.md              # Quick reference
│   └── EXAMPLE-SESSION.md         # Example output
│
├── helm_charts/                   # Kubernetes deployment
│   ├── deploy.sh                  # Phase 4: Deploy to K8s
│   ├── elasticsearch/             # Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml            # Configuration
│   │   └── templates/
│   │       ├── statefulset.yaml   # Main deployment
│   │       ├── service.yaml       # Services
│   │       ├── configmap.yaml     # ES config
│   │       └── NOTES.txt          # Post-install info
│   ├── README.md                  # Helm chart docs
│   └── QUICKSTART.md              # Quick reference
│
├── DEPLOYMENT-WORKFLOW.md         # This file
└── README.md                      # Project overview
```

## Common Scenarios

### Scenario 1: Single Air-gapped Machine

If registry and Kubernetes are on the same machine:

1. Collect images on internet-connected machine
2. Transfer images to air-gapped machine
3. Run `./epr.sh` to deploy registry
4. Run `./deploy.sh` to deploy to Kubernetes
5. Access via port-forward

### Scenario 2: Separate Registry and Kubernetes

If Kubernetes cluster nodes need to access registry on different machine:

1. Collect images on internet-connected machine
2. Transfer to registry host
3. Run `./epr.sh` on registry host
4. Configure registry to be accessible from cluster nodes:
   ```bash
   # In epr.sh or docker run, change:
   docker run -d -p 5000:5000 ... # binds to all interfaces
   ```
5. Update `values.yaml` to use registry host IP:
   ```yaml
   image:
     registry: 192.168.1.10:5000  # Registry host IP
   ```
6. Configure insecure registry on all cluster nodes
7. Deploy to Kubernetes

### Scenario 3: Multiple Kubernetes Clusters

To deploy to multiple clusters using same registry:

1. Set up registry once (Phase 1-3)
2. For each cluster:
   ```bash
   # Switch kubectl context
   kubectl config use-context cluster-1

   # Deploy
   helm install elasticsearch ./elasticsearch -n elastic
   ```

## Troubleshooting

### Images not pulling from registry

**Check from cluster node:**
```bash
# SSH to cluster node
curl http://localhost:5000/v2/_catalog

# If fails, check Docker config
cat /etc/docker/daemon.json
# Should have: {"insecure-registries": ["localhost:5000"]}

# Restart Docker if needed
sudo systemctl restart docker
```

### Pod stuck in Pending

```bash
# Check events
kubectl describe pod -n elastic elasticsearch-master-0

# Common causes:
# 1. No available nodes with enough resources
# 2. PVC not bound - check storage class
# 3. Image pull errors - check registry accessibility
```

### Registry container not starting

```bash
# Check Docker logs
docker logs elastic-registry

# Check if port is already in use
netstat -tulpn | grep 5000

# Remove and restart
docker rm -f elastic-registry
./epr.sh
```

## Scaling and Production

### Scale to 3 nodes

```bash
helm upgrade elasticsearch ./elasticsearch \
  --namespace elastic \
  --set replicas=3
```

### Enable multi-node discovery

Edit `elasticsearch/values.yaml`:
```yaml
esConfig:
  elasticsearch.yml: |
    cluster.name: elasticsearch
    discovery.seed_hosts:
      - elasticsearch-master-headless
    cluster.initial_master_nodes:
      - elasticsearch-master-0
      - elasticsearch-master-1
      - elasticsearch-master-2
```

### Increase resources

Edit `elasticsearch/values.yaml`:
```yaml
resources:
  requests:
    memory: "4Gi"
  limits:
    memory: "8Gi"

esJavaOpts: "-Xmx4g -Xms4g"
```

## Maintenance

### Update images

1. Collect new images on internet-connected machine
2. Transfer to air-gapped machine
3. Run `./epr.sh --load-only` to add to existing registry
4. Update `values.yaml` with new tag
5. Upgrade deployment:
   ```bash
   helm upgrade elasticsearch ./elasticsearch -n elastic
   ```

### Backup and restore

```bash
# Backup PVCs
kubectl get pvc -n elastic
# Use your storage provider's snapshot feature

# Or backup via Elasticsearch snapshots
# Configure snapshot repository and use _snapshot API
```

### Clean up

```bash
# Remove Kubernetes deployment
helm uninstall elasticsearch -n elastic
kubectl delete pvc -n elastic -l app=elasticsearch

# Remove registry
cd epr_deployment
./nuke_registry.sh
```

## Next Steps

1. **Deploy Kibana** - Create similar Helm chart for Kibana
2. **Deploy Logstash** - Add data pipeline
3. **Configure Ingress** - External access to services
4. **Enable Security** - X-Pack security, TLS, authentication
5. **Set up Monitoring** - Elastic Stack monitoring
6. **Configure Backups** - Snapshot repository for disaster recovery

## Support and Documentation

- EPR Deployment: See [epr_deployment/README.md](epr_deployment/README.md)
- Helm Chart: See [helm_charts/README.md](helm_charts/README.md)
- Quick Starts:
  - [epr_deployment/QUICKSTART.md](epr_deployment/QUICKSTART.md)
  - [helm_charts/QUICKSTART.md](helm_charts/QUICKSTART.md)
