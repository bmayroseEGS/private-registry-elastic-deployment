# collect-all.sh - Complete Air-gapped Collection Tool

## Overview

`collect-all.sh` is an enhanced version of `collect-images.sh` that collects **both**:
1. Container images (Elasticsearch, Kibana, Logstash, etc.)
2. k3s components (for Kubernetes installation)

This is the **recommended script** for complete air-gapped deployments.

**Can be re-run multiple times** to add more container images to your collection without re-downloading existing images.

## What It Collects

### Container Images
- Elastic Stack images (elasticsearch, kibana, logstash, etc.)
- Docker Registry image (`registry:2`)
- Any other container images you specify

### k3s Components (Optional)
- `k3s` binary (lightweight Kubernetes distribution)
- `k3s-airgap-images-amd64.tar.gz` (required container images for k3s)
- `install-k3s.sh` (installation script)
- `INSTALL-K3S.md` (installation instructions)

## Usage

### On Internet-Connected Machine

```bash
cd deployment_infrastructure
./collect-all.sh
```

The script will:
1. Check prerequisites (Docker, internet, disk space)
2. Prompt for container image URLs
3. Pull and save each image as .tar file
4. Ask if you want to download k3s components
5. Display summary and next steps

**Note:** You can re-run this script multiple times to add more images. Already-downloaded images will be skipped.

### Example Session

```
Air-gapped Collection Tool
Collects container images and k3s components

========================================
Checking Prerequisites
========================================

[INFO] Docker: Docker version 24.0.7
[INFO] Available disk space: 45GB
[INFO] Internet connectivity: OK

========================================
Creating Output Directories
========================================

[INFO] Created: ./images (for container images)
[INFO] Created: ./k3s-files (for k3s components)

========================================
Collecting Container Image URLs
========================================

[INFO] Enter container image URLs (one per line)
[INFO] When finished, just press Enter on an empty line

Image URL [1]: docker.elastic.co/elasticsearch/elasticsearch:9.2.2
[INFO] ✓ Added: docker.elastic.co/elasticsearch/elasticsearch:9.2.2

Image URL [2]: docker.elastic.co/kibana/kibana:9.2.2
[INFO] ✓ Added: docker.elastic.co/kibana/kibana:9.2.2

Image URL [3]: registry:2
[INFO] ✓ Added: registry:2

Image URL [4]:

[INFO] Finished collecting images.
[INFO] Total images to collect: 3

========================================
Pulling and Saving Container Images
========================================

[INFO] [1/3] Pulling: docker.elastic.co/elasticsearch/elasticsearch:9.2.2
[INFO] [1/3] Saving to: ./images/docker.elastic.co_elasticsearch_elasticsearch_9.2.2.tar
[INFO] [1/3] ✓ Successfully saved

...

========================================
Downloading k3s Components
========================================

[INFO] k3s is a lightweight Kubernetes distribution
[INFO] These files are needed for air-gapped k3s installation

Download k3s components for version v1.30.0+k3s1? (y/n): y
[INFO] Downloading k3s binary...
[INFO] ✓ k3s binary downloaded
[INFO] Downloading k3s airgap images...
[INFO] ✓ k3s airgap images downloaded
[INFO] Downloading k3s install script...
[INFO] ✓ k3s install script downloaded
[INFO] ✓ Installation instructions created: INSTALL-K3S.md
```

## Output Structure

After running, you'll have:

```
deployment_infrastructure/
├── collect-all.sh
├── install-k3s-airgap.sh                      # Automated k3s install script
├── uninstall-k3s-complete.sh                  # Complete k3s removal
├── k3s-files/                                 # k3s components
│   ├── k3s                                    # k3s binary
│   └── k3s-airgap-images-amd64.tar           # Required images
└── helm-files/                                # Helm binary
    └── helm

epr_deployment/
└── images/                                    # Container images
    ├── docker.elastic.co_elasticsearch_elasticsearch_9.2.2.tar
    ├── docker.elastic.co_kibana_kibana_9.2.2.tar
    ├── docker.elastic.co_logstash_logstash_9.2.2.tar
    └── registry_2.tar
```

## Transfer to Air-gapped Machine

### Option A: USB Drive
```bash
# Copy entire deployment_infrastructure folder to USB
cp -r deployment_infrastructure/ /media/usb/

# On air-gapped machine
cp -r /media/usb/deployment_infrastructure/ ~/private-registry-elastic-deployment/
```

### Option B: SCP (limited network)
```bash
# Transfer entire deployment_infrastructure folder
scp -r deployment_infrastructure/ user@airgapped-machine:~/private-registry-elastic-deployment/
```

## Installation on Air-gapped Machine

### Step 1: Install k3s (Automated)

```bash
cd deployment_infrastructure
./install-k3s-airgap.sh
# Fully automated script that:
# - Installs k3s binary and airgap images
# - Configures kubeconfig with permissions
# - Sets up localhost:5000 registry support
# - Verifies installation

# Reload shell
source ~/.bashrc

# Verify
kubectl get nodes
```

### Step 2: Deploy Local Registry

```bash
cd ../epr_deployment
./epr.sh
# Registry will be deployed at localhost:5000 with all your images
```

### Step 3: Deploy Elastic Stack to Kubernetes

```bash
cd ../helm_charts
./deploy.sh
# Interactively prompts:
# - Deploy Elasticsearch? (y/n)
# - Deploy Kibana? (y/n)
# - Deploy Logstash? (y/n)
```

### Step 4: Access Services

**Elasticsearch:**
```bash
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200
curl http://localhost:9200
```

**Kibana (via SSH tunnel from local machine):**
```bash
# On local machine
ssh -i "your-key.pem" -L 5601:localhost:5601 ubuntu@server-ip

# On remote server
kubectl port-forward -n elastic svc/kibana 5601:5601

# Open browser: http://localhost:5601
```

**Logstash:**
```bash
kubectl port-forward -n elastic svc/logstash 8080:8080
curl -X POST http://localhost:8080 -H 'Content-Type: application/json' -d '{"message":"test"}'
```

## Common Image URLs

### Elastic Stack
```
docker.elastic.co/elasticsearch/elasticsearch:9.2.2
docker.elastic.co/kibana/kibana:9.2.2
docker.elastic.co/logstash/logstash:9.2.2
docker.elastic.co/beats/elastic-agent:9.2.2
docker.elastic.co/beats/filebeat:9.2.2
docker.elastic.co/beats/metricbeat:9.2.2
```

### Required for Registry
```
registry:2
```

### Optional (for testing)
```
busybox:latest
nginx:latest
```

## Comparison: collect-all.sh vs collect-images.sh

| Feature | collect-images.sh | collect-all.sh |
|---------|------------------|----------------|
| Container images | ✅ | ✅ |
| k3s components | ❌ | ✅ |
| Installation docs | ❌ | ✅ |
| Complete solution | Partial | Complete |
| Recommended for | Images only | Full deployment |

## Troubleshooting

### "Not enough disk space"
- Check available space: `df -h .`
- Clean up: `docker system prune -a`
- Need at least 20GB free

### "Failed to download k3s"
- Check internet: `ping github.com`
- Try again (GitHub might be temporarily down)
- Or skip k3s download and install manually later

### "Failed to pull image"
- Check image URL is correct
- Check Docker Hub/registry is accessible
- Some images require authentication

## Advanced Usage

### Specify Custom k3s Version

Edit the script and change:
```bash
K3S_VERSION="v1.30.0+k3s1"  # Change to desired version
```

### Skip k3s Download

When prompted:
```
Download k3s components for version v1.30.0+k3s1? (y/n): n
```

### Collect Only Specific Images

Just enter the images you need when prompted, skip the rest.

### Re-running to Add More Images

You can run `collect-all.sh` multiple times to add more images:

```bash
cd deployment_infrastructure
./collect-all.sh
# Enter new image URLs
# Already-downloaded images will be skipped
# New images will be added to images/ directory
```

This is useful when:
- Adding new Elastic Stack components or plugins
- Updating to newer versions
- Including additional monitoring tools
- Adding custom applications to your air-gapped environment

## See Also

- [collect-images.sh](collect-images.sh) - Simple image collection only
- [epr.sh](epr.sh) - Registry deployment script
- [../helm_charts/deploy.sh](../helm_charts/deploy.sh) - Kubernetes deployment
- [../DEPLOYMENT-WORKFLOW.md](../DEPLOYMENT-WORKFLOW.md) - Complete workflow guide
