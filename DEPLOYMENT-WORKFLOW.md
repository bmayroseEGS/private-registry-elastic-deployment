# Complete Air-gapped Elastic Stack Deployment Workflow

This guide shows the complete workflow from collecting binaries and images to deploying the full Elastic Stack (Elasticsearch, Kibana, Logstash) on Kubernetes in an air-gapped environment.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT WORKFLOW                               │
└─────────────────────────────────────────────────────────────────────────┘

PHASE 1: Collection (Internet-connected Machine)
┌─────────────────────────────────────────────────────────────┐
│  Internet-connected Machine                                 │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./deployment_infrastructure/           │          │
│  │       collect-all.sh                         │          │
│  │                                               │          │
│  │  → Downloads k3s binary + airgap images      │          │
│  │  → Downloads Helm binary                     │          │
│  │  → Pulls Elastic Stack container images      │          │
│  │  → Saves everything as files                 │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Output:                                     │          │
│  │  - k3s-files/k3s (binary)                    │          │
│  │  - k3s-files/k3s-airgap-images-amd64.tar     │          │
│  │  - helm-files/helm (binary)                  │          │
│  │  - ../epr_deployment/images/*.tar            │          │
│  │    • elasticsearch-9.2.2.tar                 │          │
│  │    • kibana-9.2.2.tar                        │          │
│  │    • logstash-9.2.2.tar                      │          │
│  │    • registry-2.tar                          │          │
│  └───────────────┬──────────────────────────────┘          │
└──────────────────┼──────────────────────────────────────────┘
                   │
                   │ Transfer entire project via USB/SCP
                   ▼

PHASE 2: Infrastructure Setup (Air-gapped Machine)
┌─────────────────────────────────────────────────────────────┐
│  Air-gapped Machine                                         │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./deployment_infrastructure/           │          │
│  │       install-k3s-airgap.sh                  │          │
│  │                                               │          │
│  │  → Installs k3s from local binary            │          │
│  │  → Loads k3s airgap images                   │          │
│  │  → Configures kubeconfig                     │          │
│  │  → Starts Kubernetes cluster                 │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Kubernetes Cluster (k3s) Running            │          │
│  │  kubectl configured at:                      │          │
│  │  /etc/rancher/k3s/k3s.yaml                   │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
                   │
                   ▼

PHASE 3: Registry Deployment (Air-gapped Machine)
┌─────────────────────────────────────────────────────────────┐
│  Air-gapped Machine                                         │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./epr_deployment/epr.sh                │          │
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
                   ▼

PHASE 4: Elastic Stack Deployment
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster (k3s)                                   │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │  Run: ./helm_charts/deploy.sh                │          │
│  │                                               │          │
│  │  → Interactive prompts:                      │          │
│  │    • Deploy Elasticsearch? (y/n)             │          │
│  │    • Deploy Kibana? (y/n)                    │          │
│  │    • Deploy Logstash? (y/n)                  │          │
│  │  → Creates namespace 'elastic'               │          │
│  │  → Deploys selected components               │          │
│  │  → Waits for pods to be ready                │          │
│  └───────────────┬──────────────────────────────┘          │
│                  │                                          │
│                  ▼                                          │
│  ┌──────────────────────────────────────────────┐          │
│  │  Namespace: elastic                          │          │
│  │                                               │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  StatefulSet: elasticsearch-master     │  │          │
│  │  │  • Pod: elasticsearch-master-0         │  │          │
│  │  │  • Image: localhost:5000/              │  │          │
│  │  │           elasticsearch:9.2.2          │  │          │
│  │  │  • Ports: 9200, 9300                   │  │          │
│  │  │  • Volume: 30Gi PVC                    │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  │                                               │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  Deployment: kibana                    │  │          │
│  │  │  • Pod: kibana-xxxx                    │  │          │
│  │  │  • Image: localhost:5000/kibana:9.2.2  │  │          │
│  │  │  • Port: 5601                          │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  │                                               │          │
│  │  ┌────────────────────────────────────────┐  │          │
│  │  │  StatefulSet: logstash                 │  │          │
│  │  │  • Pod: logstash-0                     │  │          │
│  │  │  • Image: localhost:5000/              │  │          │
│  │  │           logstash:9.2.2               │  │          │
│  │  │  • Ports: 5044, 8080, 9600             │  │          │
│  │  └────────────────────────────────────────┘  │          │
│  │                                               │          │
│  │  Services:                                    │          │
│  │  - elasticsearch-master (ClusterIP:9200)      │          │
│  │  - kibana (ClusterIP:5601)                    │          │
│  │  - logstash (ClusterIP:5044,8080,9600)        │          │
│  └──────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

## Step-by-Step Workflow

### Prerequisites

**Internet-connected Machine:**
- Docker installed (for pulling container images)
- Internet access to GitHub, Docker Hub, and Elastic registries
- curl and wget installed

**Air-gapped Machine:**
- Docker installed and running
- Ubuntu/Debian-based Linux (or compatible system for k3s)
- Root/sudo access
- At least 4GB RAM, 2 CPU cores
- 50GB+ free disk space

### Phase 1: Collect All Binaries & Images (Internet-connected)

1. **Clone the repository**
   ```bash
   git clone <repo-url>
   cd private-registry-elastic-deployment
   ```

2. **Run the collection script**
   ```bash
   cd deployment_infrastructure
   ./collect-all.sh
   ```

3. **Script automatically downloads:**
   - k3s binary (Kubernetes)
   - k3s airgap images (for offline installation)
   - Helm binary (package manager)
   - Elastic Stack container images:
     - Elasticsearch 9.2.2
     - Kibana 9.2.2
     - Logstash 9.2.2
     - Docker Registry 2

4. **Enter additional image URLs if prompted**
   ```
   Do you want to add more images? (y/n): y
   Image URL [5]: docker.io/some-custom-image:latest
   Image URL [6]: [press Enter to finish]
   ```

5. **Wait for completion**
   - Downloads binaries to k3s-files/ and helm-files/
   - Pulls and saves images to ../epr_deployment/images/
   - May take 20-45 minutes depending on connection speed

6. **Verify output**
   ```bash
   # Check k3s files
   ls -lh k3s-files/
   # Should show: k3s, k3s-airgap-images-amd64.tar, install.sh

   # Check Helm
   ls -lh helm-files/
   # Should show: helm

   # Check images
   ls -lh ../epr_deployment/images/
   # Should show: .tar files for each image
   ```

### Phase 2: Transfer to Air-gapped Machine

Transfer the entire project directory to preserve structure and paths.

**Option A: USB Drive**
```bash
# On internet-connected machine
cd ..
tar -czf private-registry-elastic-deployment.tar.gz private-registry-elastic-deployment/
cp private-registry-elastic-deployment.tar.gz /media/usb/

# On air-gapped machine
cp /media/usb/private-registry-elastic-deployment.tar.gz ~/
cd ~
tar -xzf private-registry-elastic-deployment.tar.gz
cd private-registry-elastic-deployment
```

**Option B: SCP (if limited network access)**
```bash
# Transfer entire project
cd ..
scp -r private-registry-elastic-deployment/ user@airgapped:~/
```

### Phase 2.5: Setup Infrastructure (Air-gapped)

1. **Navigate to project directory**
   ```bash
   cd ~/private-registry-elastic-deployment
   ```

2. **Optional: Run setup script**
   ```bash
   ./deployment_infrastructure/setup-machine.sh
   # Installs essential tools and configures system
   ```

3. **Install k3s Kubernetes**
   ```bash
   ./deployment_infrastructure/install-k3s-airgap.sh
   ```

   The script will:
   - Install k3s binary to /usr/local/bin/
   - Load k3s airgap images
   - Configure kubeconfig at /etc/rancher/k3s/k3s.yaml
   - Create kubectl symlink
   - Start k3s service
   - Verify cluster is running

4. **Verify k3s installation**
   ```bash
   kubectl get nodes
   # Should show: Ready

   kubectl get pods -A
   # Should show system pods running
   ```

5. **Verify Helm installation**
   ```bash
   helm version
   # Should show installed version
   ```

### Phase 3: Deploy Registry (Air-gapped)

1. **Navigate to deployment directory**
   ```bash
   cd ~/private-registry-elastic-deployment/epr_deployment
   ```

2. **Verify images are present**
   ```bash
   ls -lh images/
   # Should show: elasticsearch, kibana, logstash, registry .tar files
   ```

3. **Run the EPR deployment script**
   ```bash
   ./epr.sh
   ```

4. **Follow prompts**
   - Configure Docker insecure registry (auto or manual)
   - Wait for registry container to start
   - Wait for images to load and push to localhost:5000

5. **Verify registry**
   ```bash
   curl http://localhost:5000/v2/_catalog
   # Should show: {"repositories":["elasticsearch","kibana","logstash"]}

   # Check specific images
   curl http://localhost:5000/v2/elasticsearch/tags/list
   curl http://localhost:5000/v2/kibana/tags/list
   curl http://localhost:5000/v2/logstash/tags/list
   ```

### Phase 4: Deploy Elastic Stack to Kubernetes

1. **Navigate to helm charts**
   ```bash
   cd ../helm_charts
   ```

2. **Review configuration** (optional)
   ```bash
   # Check Elasticsearch config
   cat elasticsearch/values.yaml

   # Check Kibana config
   cat kibana/values.yaml

   # Check Logstash config
   cat logstash/values.yaml
   ```

3. **Deploy using interactive script**
   ```bash
   ./deploy.sh
   ```

   The script will prompt:
   ```
   Deploy Elasticsearch? (y/n): y
   Deploy Kibana? (y/n): y
   Deploy Logstash? (y/n): y
   ```

   For each selected component, the script will:
   - Check prerequisites (kubectl, helm, registry)
   - Create namespace 'elastic' if needed
   - Install Helm chart from local registry
   - Wait for pods to be ready (timeout: 10m)
   - Display deployment progress with spinner

4. **Or deploy components manually**
   ```bash
   # Deploy Elasticsearch
   helm install elasticsearch ./elasticsearch \
     --namespace elastic \
     --create-namespace \
     --wait \
     --timeout 10m

   # Deploy Kibana
   helm install kibana ./kibana \
     --namespace elastic \
     --wait \
     --timeout 10m

   # Deploy Logstash
   helm install logstash ./logstash \
     --namespace elastic \
     --wait \
     --timeout 10m
   ```

5. **Verify deployment**
   ```bash
   # Check all resources
   kubectl get all -n elastic

   # Check pods status
   kubectl get pods -n elastic
   # Should show:
   # elasticsearch-master-0   1/1  Running
   # kibana-xxxx              1/1  Running
   # logstash-0               1/1  Running

   # Check services
   kubectl get svc -n elastic

   # Check persistent volumes
   kubectl get pvc -n elastic
   ```

### Phase 5: Access Services

#### Elasticsearch

1. **Port-forward (on remote server)**
   ```bash
   kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
   ```

2. **Test connection**
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

#### Kibana

**From Remote Server (Direct):**
```bash
kubectl port-forward -n elastic svc/kibana 5601:5601
curl http://localhost:5601
```

**From Local Machine (via SSH Tunnel):**
```bash
# Step 1: Create SSH tunnel from local to remote
ssh -i "your-key.pem" -L 5601:localhost:5601 ubuntu@your-server-ip

# Step 2: On remote server, port-forward Kibana
kubectl port-forward -n elastic svc/kibana 5601:5601

# Step 3: Open browser on local machine
# Navigate to: http://localhost:5601
```

#### Logstash

1. **Port-forward HTTP input**
   ```bash
   kubectl port-forward -n elastic svc/logstash 8080:8080
   ```

2. **Send test event**
   ```bash
   curl -X POST http://localhost:8080 \
     -H 'Content-Type: application/json' \
     -d '{"message":"test event","source":"manual"}'
   ```

3. **Check Logstash monitoring**
   ```bash
   kubectl port-forward -n elastic svc/logstash 9600:9600
   curl http://localhost:9600/_node/stats?pretty
   ```

4. **View logs**
   ```bash
   kubectl logs -n elastic -l app=logstash -f
   ```

## Directory Structure

```
private-registry-elastic-deployment/
├── deployment_infrastructure/      # Infrastructure setup
│   ├── collect-all.sh             # Phase 1: Collect everything
│   ├── install-k3s-airgap.sh      # Phase 2.5: Install k3s
│   ├── setup-machine.sh           # Optional: System setup
│   ├── uninstall-k3s-complete.sh  # Cleanup k3s
│   ├── k3s-files/                 # k3s binaries (created by collect-all)
│   ├── helm-files/                # Helm binary (created by collect-all)
│   └── *.md                       # Documentation
│
├── epr_deployment/                 # Registry deployment
│   ├── epr.sh                     # Phase 3: Deploy registry
│   ├── nuke_registry.sh           # Cleanup registry
│   ├── images/                    # Container images (gitignored)
│   └── *.md                       # Documentation
│
├── helm_charts/                    # Kubernetes deployment
│   ├── deploy.sh                  # Phase 4: Deploy Elastic Stack
│   │
│   ├── elasticsearch/             # Elasticsearch Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── statefulset.yaml
│   │       ├── service.yaml
│   │       ├── configmap.yaml
│   │       └── NOTES.txt
│   │
│   ├── kibana/                    # Kibana Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── configmap.yaml
│   │       └── NOTES.txt
│   │
│   ├── logstash/                  # Logstash Helm chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── statefulset.yaml
│   │       ├── service.yaml
│   │       ├── configmap-config.yaml
│   │       ├── configmap-pipeline.yaml
│   │       └── NOTES.txt
│   │
│   └── *.md                       # Documentation
│
├── DEPLOYMENT-WORKFLOW.md          # This file
└── README.md                       # Project overview
```

## Common Scenarios

### Scenario 1: Single Air-gapped Machine (Most Common)

Complete deployment on one air-gapped server:

1. **Internet machine:** Run `./deployment_infrastructure/collect-all.sh`
2. **Transfer:** Copy entire project to air-gapped machine
3. **Air-gapped machine:**
   - Install k3s: `./deployment_infrastructure/install-k3s-airgap.sh`
   - Deploy registry: `./epr_deployment/epr.sh`
   - Deploy stack: `./helm_charts/deploy.sh`
4. **Access:** Use kubectl port-forward and SSH tunnels

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

After successfully deploying the stack, consider these enhancements:

1. **Configure Ingress** - Expose services externally with Ingress controller
2. **Enable Security** - X-Pack security, TLS certificates, authentication
3. **Set up Monitoring** - Enable Elastic Stack monitoring features
4. **Configure Backups** - Elasticsearch snapshot repository
5. **Deploy Beats** - Add Filebeat, Metricbeat for data collection
6. **Scale Resources** - Increase replicas and resources for production
7. **Add Custom Plugins** - Collect additional images and deploy plugins
8. **Network Policies** - Restrict pod-to-pod communication

## Support and Documentation

- EPR Deployment: See [epr_deployment/README.md](epr_deployment/README.md)
- Helm Chart: See [helm_charts/README.md](helm_charts/README.md)
- Quick Starts:
  - [epr_deployment/QUICKSTART.md](epr_deployment/QUICKSTART.md)
  - [helm_charts/QUICKSTART.md](helm_charts/QUICKSTART.md)
