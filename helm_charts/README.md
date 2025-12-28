# Elasticsearch Helm Chart for Air-gapped Deployment

This Helm chart deploys Elasticsearch to a Kubernetes cluster using images from a local container registry.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Namespace: elastic                     │ │
│  │                                                     │ │
│  │  ┌──────────────────────────────────────────────┐  │ │
│  │  │         StatefulSet: elasticsearch           │  │ │
│  │  │                                              │  │ │
│  │  │  ┌────────────┐  ┌────────────┐             │  │ │
│  │  │  │   Pod 1    │  │   Pod 2    │   ...       │  │ │
│  │  │  │            │  │            │             │  │ │
│  │  │  │  ES:9200   │  │  ES:9200   │             │  │ │
│  │  │  │  ES:9300   │  │  ES:9300   │             │  │ │
│  │  │  └─────┬──────┘  └─────┬──────┘             │  │ │
│  │  │        │               │                    │  │ │
│  │  │        └───────┬───────┘                    │  │ │
│  │  └────────────────┼────────────────────────────┘  │ │
│  │                   │                               │ │
│  │         ┌─────────┴─────────┐                     │ │
│  │         │                   │                     │ │
│  │  ┌──────▼───────┐    ┌─────▼──────┐              │ │
│  │  │  Service     │    │ Headless   │              │ │
│  │  │ ClusterIP    │    │  Service   │              │ │
│  │  │   :9200      │    │ (StatefulSet)             │ │
│  │  └──────────────┘    └────────────┘              │ │
│  │                                                   │ │
│  │  ┌──────────────────────────────────────┐        │ │
│  │  │  PersistentVolumeClaims (PVC)        │        │ │
│  │  │  - data-elasticsearch-master-0       │        │ │
│  │  │  - data-elasticsearch-master-1       │        │ │
│  │  └──────────────────────────────────────┘        │ │
│  └───────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
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
3. **Local registry** - Running with Elasticsearch images loaded (see [../epr_deployment](../epr_deployment))
4. **kubectl access** - Configured to access your cluster

## Quick Start

### 1. Deploy using the script

```bash
cd helm_charts
./deploy.sh
```

The script will:
- Check prerequisites (kubectl, helm, cluster access)
- Verify local registry is accessible
- Create namespace
- Deploy Elasticsearch
- Wait for pods to be ready
- Display status and access instructions

### 2. Manual Deployment

```bash
# Create namespace
kubectl create namespace elastic

# Install the chart
helm install elasticsearch ./elasticsearch \
  --namespace elastic \
  --wait \
  --timeout 10m

# Check status
kubectl get pods -n elastic
```

## Configuration

### Key Configuration Options

Edit [values.yaml](elasticsearch/values.yaml) to customize:

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

#### Node Roles
```yaml
roles:
  - master  # Can be elected master
  - data    # Stores data
  - ingest  # Processes documents
```

## Accessing Elasticsearch

### Port-forward Method (Development)

```bash
# Forward port 9200 to localhost
kubectl port-forward -n elastic svc/elasticsearch-master 9200:9200

# In another terminal
curl http://localhost:9200
curl http://localhost:9200/_cluster/health?pretty
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
kubectl get pods -n elastic -l app=elasticsearch
```

### View Logs
```bash
# All pods
kubectl logs -n elastic -l app=elasticsearch -f

# Specific pod
kubectl logs -n elastic elasticsearch-master-0 -f
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
# Remove the Helm release
helm uninstall elasticsearch -n elastic

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

1. **Deploy Kibana** - Create similar chart for Kibana
2. **Deploy Logstash** - Add data processing pipeline
3. **Configure Backups** - Set up snapshot repository
4. **Enable Monitoring** - Use Elastic Stack monitoring
5. **Production Hardening** - Enable security, TLS, RBAC

## File Structure

```
elasticsearch/
├── Chart.yaml              # Chart metadata
├── values.yaml            # Default configuration
├── templates/
│   ├── statefulset.yaml   # Main Elasticsearch deployment
│   ├── service.yaml       # Services (ClusterIP + Headless)
│   ├── configmap.yaml     # Elasticsearch configuration
│   └── NOTES.txt          # Post-install instructions
└── charts/                # Dependencies (none currently)
```

## Support

For issues or questions:
1. Check the [troubleshooting section](#monitoring-and-troubleshooting)
2. Review Elasticsearch logs: `kubectl logs -n elastic -l app=elasticsearch`
3. Check Kubernetes events: `kubectl get events -n elastic`
