---
sidebar_position: 5
---

# Kubernetes Integration

Raworc provides seamless Kubernetes integration to create isolated execution environments for each session. When enabled, each session runs in its own pod with persistent storage, providing security, isolation, and scalability.

## Overview

The Kubernetes integration allows Raworc to:
- Create isolated containers for each session
- Provide persistent storage via PersistentVolumeClaims
- Manage resource limits and quotas
- Automatically clean up resources when sessions are deleted
- Support both in-cluster and external kubeconfig authentication

## Prerequisites

Before enabling Kubernetes integration, ensure you have:

1. **A Kubernetes cluster** (1.24+ recommended)
2. **kubectl configured** with access to your cluster
3. **A namespace** for Raworc sessions
4. **RBAC permissions** to create pods and PVCs

## Configuration

### Environment Variables

Configure Kubernetes integration in your `.env` file:

```env
# Enable/Disable Kubernetes Integration
KUBERNETES_ENABLED=true

# Kubernetes Namespace for Sessions
KUBERNETES_NAMESPACE=raworc-sessions

# Container Image for Sessions
KUBERNETES_IMAGE=python:3.11-slim
KUBERNETES_IMAGE_PULL_POLICY=IfNotPresent

# Resource Requests (minimum guaranteed)
KUBERNETES_CPU_REQUEST=100m
KUBERNETES_MEMORY_REQUEST=128Mi

# Resource Limits (maximum allowed)
KUBERNETES_CPU_LIMIT=1000m
KUBERNETES_MEMORY_LIMIT=1Gi

# Persistent Storage Size
KUBERNETES_STORAGE_SIZE=1Gi

# Optional: Storage Class
# KUBERNETES_STORAGE_CLASS=standard
```

### Namespace Setup

Create the namespace for Raworc sessions:

```bash
kubectl create namespace raworc-sessions
```

Verify the namespace:

```bash
kubectl get namespace raworc-sessions
```

## How It Works

### Session Creation Flow

When a session is created with Kubernetes enabled:

1. **PersistentVolumeClaim Creation**
   - A PVC is created with the configured storage size
   - Named: `session-pvc-{session-id}`
   - Provides persistent workspace at `/workspace`

2. **Pod Creation**
   - A pod is created with the configured image
   - Named: `session-{session-id}`
   - Mounts the PVC at `/workspace`
   - Injects session environment variables

3. **Container Initialization**
   - Container starts with a keep-alive process
   - Working directory set to `/workspace`
   - Ready to execute session workloads

### Resource Management

Each session pod is configured with:

```yaml
resources:
  requests:
    cpu: "100m"      # Minimum CPU
    memory: "128Mi"  # Minimum memory
  limits:
    cpu: "1000m"     # Maximum CPU
    memory: "1Gi"    # Maximum memory
```

### Labels and Metadata

Pods are labeled for easy identification:

```yaml
labels:
  app: raworc
  component: session
  session-id: <uuid>
  session-name: <session-name>
```

## API Usage

### Creating a Session with Kubernetes

```bash
curl -X POST http://localhost:9000/api/v0/sessions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ml-training-session",
    "namespace": "default",
    "starting_prompt": "ML model training environment",
    "agent_ids": []
  }'
```

Response includes container information:

```json
{
  "id": "66709101-b7dd-460f-95a3-799177797a5c",
  "name": "ml-training-session",
  "container_id": "3e2b416e-ed22-48f2-986e-af825f270a2e",
  "persistent_volume_id": "pvc-887e4806-34e6-4021-bb19-f7ea01ced357",
  ...
}
```

### Monitoring Session Resources

Check pod status:

```bash
kubectl get pod session-<session-id> -n raworc-sessions
```

View pod details:

```bash
kubectl describe pod session-<session-id> -n raworc-sessions
```

Check storage usage:

```bash
kubectl exec session-<session-id> -n raworc-sessions -- df -h /workspace
```

### Session Deletion

When a session is deleted via the API:

```bash
curl -X DELETE http://localhost:9000/api/v0/sessions/{session-id} \
  -H "Authorization: Bearer $TOKEN"
```

Raworc automatically:
1. Deletes the pod (graceful termination)
2. Deletes the PVC (data cleanup)
3. Removes database records

## Authentication Methods

### In-Cluster Authentication

When Raworc runs inside Kubernetes:
- Automatically uses ServiceAccount credentials
- No additional configuration needed
- Recommended for production deployments

### Kubeconfig Authentication

When Raworc runs outside Kubernetes:
- Uses `~/.kube/config` or `KUBECONFIG` environment variable
- Suitable for development and testing
- Requires kubectl to be configured

## Security Considerations

### Pod Security

1. **Non-root containers**: Configure pods to run as non-root user
2. **Read-only root filesystem**: Mount root filesystem as read-only
3. **Security contexts**: Apply appropriate security contexts

Example security configuration:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
```

### Network Policies

Implement network policies to restrict pod communication:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: session-pods-isolation
  namespace: raworc-sessions
spec:
  podSelector:
    matchLabels:
      component: session
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: raworc-sessions
```

### RBAC Configuration

Create a ServiceAccount with minimal permissions:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: raworc-controller
  namespace: raworc-sessions
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: raworc-controller
  namespace: raworc-sessions
rules:
- apiGroups: [""]
  resources: ["pods", "persistentvolumeclaims"]
  verbs: ["create", "get", "list", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: raworc-controller
  namespace: raworc-sessions
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: raworc-controller
subjects:
- kind: ServiceAccount
  name: raworc-controller
  namespace: raworc-sessions
```

## Monitoring and Observability

### Prometheus Metrics

Export pod metrics to Prometheus:

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: raworc-sessions
spec:
  selector:
    matchLabels:
      app: raworc
      component: session
  endpoints:
  - port: metrics
```

### Logging

View pod logs:

```bash
kubectl logs session-<session-id> -n raworc-sessions
```

Stream logs in real-time:

```bash
kubectl logs -f session-<session-id> -n raworc-sessions
```

### Resource Usage

Monitor resource consumption:

```bash
kubectl top pod session-<session-id> -n raworc-sessions
```

## Troubleshooting

### Common Issues

#### Pods Stuck in Pending

**Symptoms**: Pods remain in `Pending` state

**Solutions**:
- Check resource availability: `kubectl describe pod session-<id>`
- Verify PVC is bound: `kubectl get pvc -n raworc-sessions`
- Check node capacity: `kubectl top nodes`

#### PVC Not Binding

**Symptoms**: PVC remains in `Pending` state

**Solutions**:
- Check StorageClass exists: `kubectl get storageclass`
- Verify storage provisioner is running
- Check available storage capacity

#### Permission Denied Errors

**Symptoms**: Raworc cannot create pods/PVCs

**Solutions**:
- Verify RBAC permissions: `kubectl auth can-i create pods -n raworc-sessions`
- Check ServiceAccount configuration
- Review cluster role bindings

### Debug Commands

Enable debug logging:

```env
RUST_LOG=debug,raworc::kubernetes=trace
```

Test Kubernetes connectivity:

```bash
kubectl cluster-info
kubectl auth can-i create pods -n raworc-sessions
```

Check Raworc logs:

```bash
tail -f logs/raworc.log | grep kubernetes
```

## Best Practices

### Resource Management

1. **Set appropriate limits**: Balance between resource usage and performance
2. **Use resource quotas**: Prevent resource exhaustion
3. **Implement pod priority**: Ensure critical sessions get resources
4. **Monitor usage patterns**: Adjust limits based on actual usage

### Storage

1. **Choose appropriate storage class**: SSD for performance-critical workloads
2. **Implement backup strategies**: Regular snapshots of PVCs
3. **Set retention policies**: Clean up old PVCs periodically
4. **Monitor storage usage**: Alert on high usage

### Scaling

1. **Horizontal Pod Autoscaling**: Scale based on metrics
2. **Cluster Autoscaling**: Add nodes when needed
3. **Load distribution**: Spread sessions across nodes
4. **Resource pools**: Dedicate node pools for sessions

## Advanced Configuration

### Custom Images

Build custom images for specific workloads:

```dockerfile
FROM python:3.11-slim
RUN pip install numpy pandas scikit-learn
WORKDIR /workspace
```

Configure in `.env`:

```env
KUBERNETES_IMAGE=myregistry/custom-session:latest
```

### GPU Support

For GPU-enabled sessions:

```env
KUBERNETES_IMAGE=nvidia/cuda:11.8.0-runtime-ubuntu22.04
```

Add GPU resource requests:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

### Multi-Container Sessions

For complex workloads, modify the pod spec to include sidecar containers:

```yaml
containers:
- name: session-container
  image: python:3.11-slim
- name: database
  image: postgres:14
  env:
  - name: POSTGRES_PASSWORD
    value: session-db-pass
```

## Migration Guide

### Migrating from Non-Kubernetes to Kubernetes

1. **Prepare the cluster**: Set up namespace and RBAC
2. **Update configuration**: Set `KUBERNETES_ENABLED=true`
3. **Test with new sessions**: Create test sessions first
4. **Monitor resource usage**: Ensure cluster can handle load
5. **Migrate existing sessions**: Recreate if needed

### Rollback Procedure

If issues arise:

1. Set `KUBERNETES_ENABLED=false`
2. Restart Raworc service
3. Sessions will run without containers
4. Existing pods can be manually cleaned up

## Performance Tuning

### Pod Startup Time

Reduce startup time:
- Use smaller base images
- Pre-pull images on nodes
- Optimize init containers

### Network Latency

Minimize latency:
- Use local SSD storage
- Place pods close to Raworc
- Optimize DNS resolution

### Resource Utilization

Improve efficiency:
- Right-size resource requests
- Use vertical pod autoscaling
- Implement pod disruption budgets

## Future Enhancements

Planned improvements for Kubernetes integration:

- **StatefulSets**: For ordered, persistent sessions
- **Job scheduling**: Batch processing capabilities
- **Service mesh integration**: Advanced networking features
- **Operator pattern**: Custom resource definitions
- **Multi-cluster support**: Federated session management