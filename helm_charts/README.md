# Elastic Stack Helm Charts for Air-gapped Deployment

This directory contains Helm charts for deploying the complete Elastic Stack (Elasticsearch, Kibana, Logstash) to a Kubernetes cluster using images from a local container registry.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Namespace: elastic                             │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────┐          │ │
│  │  │  StatefulSet: elasticsearch-master           │          │ │
│  │  │  • Pod: elasticsearch-master-0               │          │ │
│  │  │  • Image: localhost:5000/elasticsearch:9.2.2 │          │ │
│  │  │  • Ports: 9200, 9300                         │          │ │
│  │  │  • Volume: 30Gi PVC                          │          │ │
│  │  └──────────────────────────────────────────────┘          │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────┐          │ │
│  │  │  Deployment: kibana                          │          │ │
│  │  │  • Pod: kibana-xxxx                          │          │ │
│  │  │  • Image: localhost:5000/kibana:9.2.2        │          │ │
│  │  │  • Port: 5601                                │          │ │
│  │  └──────────────────────────────────────────────┘          │ │
│  │                                                             │ │
│  │  ┌──────────────────────────────────────────────┐          │ │
│  │  │  StatefulSet: logstash                       │          │ │
│  │  │  • Pod: logstash-0                           │          │ │
│  │  │  • Image: localhost:5000/logstash:9.2.2      │          │ │
│  │  │  • Ports: 5044, 8080, 9600                   │          │ │
│  │  └──────────────────────────────────────────────┘          │ │
│  │                                                             │ │
│  │  Services:                                                  │ │
│  │  - elasticsearch-master (ClusterIP:9200)                    │ │
│  │  - kibana (ClusterIP:5601)                                  │ │
│  │  - logstash (ClusterIP:5044,8080,9600)                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
                         │
                         │ pulls images from
                         ▼
              ┌──────────────────────┐
              │  Local Registry      │
              │  localhost:5000      │
              │                      │
              │  - elasticsearch:9.2.2│
              │  - kibana:9.2.2      │
              │  - logstash:9.2.2    │
              └──────────────────────┘
```

## Prerequisites

1. **Kubernetes cluster** - Running and accessible via kubectl
2. **Helm 3.x** - Installed and configured
3. **Local registry** - Running with Elastic Stack images loaded (see [../epr_deployment](../epr_deployment))
4. **kubectl access** - Configured to access your cluster

## Quick Start

### 1. Deploy using the script (Recommended)

```bash
cd helm_charts
./deploy.sh
```

The script will:
- Check prerequisites (kubectl, helm, cluster access)
- Verify local registry is accessible
- Create namespace
- Interactively prompt for each component:
  - Deploy Elasticsearch? (y/n)
  - Deploy Kibana? (y/n)
  - Deploy Logstash? (y/n)
- Deploy selected components
- Wait for pods to be ready
- Display status and access instructions

### 2. Manual Deployment

```bash
# Create namespace
kubectl create namespace elastic

# Install Elasticsearch
helm install elasticsearch ./elasticsearch \
  --namespace elastic \
  --wait \
  --timeout 10m

# Install Kibana
helm install kibana ./kibana \
  --namespace elastic \
  --wait \
  --timeout 10m

# Install Logstash
helm install logstash ./logstash \
  --namespace elastic \
  --wait \
  --timeout 10m

# Check status
kubectl get pods -n elastic
```

## Configuration

### Elasticsearch Configuration

Edit [elasticsearch/values.yaml](elasticsearch/values.yaml) to customize:

#### Image Configuration
```yaml
image:
  registry: localhost:5000        # Your local registry
  repository: elasticsearch       # Image repository
  tag: "9.2.2"                   # Elasticsearch version
```

#### Cluster Size
```yaml
replicas: 1  # Start with 1, scale to 3 for production
```

#### Resources
```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"
```

#### Persistence
```yaml
persistence:
  enabled: true
  size: 30Gi
  storageClass: ""  # Use default or specify
```

### Kibana Configuration

Edit [kibana/values.yaml](kibana/values.yaml) to customize:

```yaml
image:
  registry: localhost:5000
  repository: kibana
  tag: "9.2.2"

replicas: 1

elasticsearchHosts: "http://elasticsearch-master:9200"
```

### Logstash Configuration

Edit [logstash/values.yaml](logstash/values.yaml) to customize:

```yaml
image:
  registry: localhost:5000
  repository: logstash
  tag: "9.2.2"

replicas: 1

elasticsearchHosts: "http://elasticsearch-master:9200"
```

## Accessing Services

### Elasticsearch (Port-forward Method)

```bash
# Forward port 9200 to localhost
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200

# In another terminal
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
```

### Kibana (Port-forward Method)

```bash
# Forward port 5601 to localhost
kubectl port-forward -n elastic svc/kibana 5601:5601

# Open browser: http://localhost:5601
```

### Kibana (SSH Tunnel from Local Machine)

```bash
# Step 1: Create SSH tunnel from local to remote
ssh -i "your-key.pem" -L 5601:localhost:5601 ubuntu@your-server-ip

# Step 2: On remote server, port-forward Kibana
kubectl port-forward -n elastic svc/kibana 5601:5601

# Step 3: Open browser on local machine
# Navigate to: http://localhost:5601
```

### Logstash (HTTP Input)

```bash
# Port-forward HTTP input
kubectl port-forward -n elastic svc/logstash 8080:8080

# Send test event
curl -X POST http://localhost:8080 \
  -H 'Content-Type: application/json' \
  -d '{"message":"test event","source":"manual"}'

# Check Logstash monitoring
kubectl port-forward -n elastic svc/logstash 9600:9600
curl http://localhost:9600/_node/stats?pretty
```

### NodePort Method (Testing)

Edit values.yaml:
```yaml
service:
  type: NodePort
  nodePort: 30920
```

Then upgrade:
```bash
helm upgrade elasticsearch ./elasticsearch -n elastic
```

Access via: `http://<node-ip>:30920`

### LoadBalancer Method (Production)

Edit values.yaml:
```yaml
service:
  type: LoadBalancer
```

Get the external IP:
```bash
kubectl get svc -n elastic elasticsearch-master
```

## Scaling the Cluster

### Scale to 3 nodes (recommended for production)

```bash
helm upgrade elasticsearch ./elasticsearch \
  --namespace elastic \
  --set replicas=3 \
  --wait
```

Or edit values.yaml and set `replicas: 3`, then:

```bash
helm upgrade elasticsearch ./elasticsearch -n elastic
```

### Enable Multi-node Discovery

For multi-node clusters, edit values.yaml:

```yaml
esConfig:
  elasticsearch.yml: |
    cluster.name: ${CLUSTER_NAME}
    network.host: 0.0.0.0

    # Change from single-node to multi-node
    discovery.seed_hosts:
      - elasticsearch-master-headless
    cluster.initial_master_nodes:
      - elasticsearch-master-0
      - elasticsearch-master-1
      - elasticsearch-master-2
```

## Monitoring and Troubleshooting

### Check Pod Status
```bash
# All pods in namespace
kubectl get pods -n elastic

# Specific components
kubectl get pods -n elastic -l app=elasticsearch
kubectl get pods -n elastic -l app=kibana
kubectl get pods -n elastic -l app=logstash
```

### View Logs
```bash
# Elasticsearch
kubectl logs -n elastic -l app=elasticsearch -f
kubectl logs -n elastic elasticsearch-master-0 -f

# Kibana
kubectl logs -n elastic -l app=kibana -f

# Logstash
kubectl logs -n elastic -l app=logstash -f
kubectl logs -n elastic logstash-0 -f
```

### Check Cluster Health
```bash
# Port-forward first
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200

# Then check
curl http://localhost:9200/_cluster/health?pretty
curl http://localhost:9200/_cat/nodes?v
curl http://localhost:9200/_cat/indices?v
```

### Common Issues

#### Pod stuck in Pending
```bash
# Check events
kubectl describe pod -n elastic elasticsearch-master-0

# Common causes:
# - Insufficient resources
# - PVC not bound
# - Node selector constraints
```

#### CrashLoopBackOff
```bash
# Check logs
kubectl logs -n elastic elasticsearch-master-0

# Common causes:
# - vm.max_map_count too low (should be 262144)
# - Insufficient memory
# - Configuration errors
```

#### Can't pull image
```bash
# Verify registry is accessible from cluster nodes
# On each node:
curl http://localhost:5000/v2/_catalog

# Check if insecure registry is configured in Docker
# Edit /etc/docker/daemon.json on each node
```

## Customization Examples

### Custom JVM Heap Size

```yaml
esJavaOpts: "-Xmx4g -Xms4g"  # 4GB heap
resources:
  limits:
    memory: "8Gi"  # Should be 2x heap size
```

### Add Environment Variables

```yaml
extraEnvs:
  - name: CUSTOM_VAR
    value: "custom_value"
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

### Node Affinity

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - elasticsearch
      topologyKey: kubernetes.io/hostname
```

## Uninstalling

```bash
# Remove individual components
helm uninstall elasticsearch -n elastic
helm uninstall kibana -n elastic
helm uninstall logstash -n elastic

# Or remove all at once
helm uninstall elasticsearch kibana logstash -n elastic

# Optionally delete PVCs (WARNING: deletes data!)
kubectl delete pvc -n elastic -l app=elasticsearch

# Optionally delete namespace
kubectl delete namespace elastic
```

## Security Considerations

### Current Configuration (Development)

This chart is configured for development/testing with security disabled:
```yaml
xpack.security.enabled: false
```

### For Production

1. **Enable X-Pack Security**
```yaml
esConfig:
  elasticsearch.yml: |
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
```

2. **Set passwords** - Use elasticsearch-setup-passwords
3. **Enable TLS** - Configure certificates
4. **Network policies** - Restrict pod-to-pod traffic
5. **RBAC** - Limit service account permissions

## Next Steps

1. **Scale Elasticsearch** - Increase to 3+ nodes for production
2. **Configure Backups** - Set up snapshot repository
3. **Enable Monitoring** - Use Elastic Stack monitoring
4. **Production Hardening** - Enable security, TLS, RBAC
5. **Deploy Beats** - Add Filebeat, Metricbeat for data collection
6. **Configure Ingress** - Expose services externally

## File Structure

```
helm_charts/
├── deploy.sh                  # Interactive deployment script
│
├── elasticsearch/             # Elasticsearch Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── statefulset.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── NOTES.txt
│
├── kibana/                    # Kibana Helm chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── configmap.yaml
│       └── NOTES.txt
│
└── logstash/                  # Logstash Helm chart
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── statefulset.yaml
        ├── service.yaml
        ├── configmap-config.yaml
        ├── configmap-pipeline.yaml
        └── NOTES.txt
```

## Support

For issues or questions:
1. Check the [troubleshooting section](#monitoring-and-troubleshooting)
2. Review Elasticsearch logs: `kubectl logs -n elastic -l app=elasticsearch`
3. Check Kubernetes events: `kubectl get events -n elastic`
