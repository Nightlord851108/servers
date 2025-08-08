# Memory MCP Kubernetes Deployment

This directory contains Kubernetes YAML files for deploying the Memory MCP server in different modes.

## Files

- `k8s-namespace.yaml`: Namespace definition for Memory MCP resources
- `k8s-deployment.yaml`: HTTP mode deployment with service, ingress, and persistent storage
- `k8s-stdio-deployment.yaml`: STDIO mode deployment for batch processing or integration
- `k8s-stdio-bridge.yaml`: STDIO bridge deployment for external access to STDIO mode
- `k8s-cloudflare-tunnel.yaml`: Cloudflare Tunnel configuration using tunnel token
- `k8s-cloudflare-tunnel-credentials.yaml`: Cloudflare Tunnel configuration using credentials file
- `cloudflare-tunnel-setup.md`: Complete setup guide for Cloudflare Tunnel
- `k8s-README.md`: This documentation file

## Namespace Configuration

All resources are deployed in the `mcp` namespace by default. You can customize this by:

### Method 1: Create namespace first
```bash
# Create the namespace
kubectl apply -f k8s-namespace.yaml

# Then deploy your resources
kubectl apply -f k8s-deployment.yaml
```

### Method 2: Use kubectl with custom namespace
```bash
# Deploy to custom namespace (creates namespace if not exists)
kubectl apply -f k8s-deployment.yaml -n your-custom-namespace

# Or set default namespace context
kubectl config set-context --current --namespace=your-custom-namespace
```

### Method 3: Modify YAML files
Change the `namespace` field in all YAML files from `mcp` to your desired namespace.

## HTTP Mode Deployment

The HTTP mode is suitable for web applications and services that need to interact with the MCP server via HTTP API.

### Features
- **Service**: ClusterIP service exposing port 80
- **Ingress**: NGINX ingress for external access
- **Persistent Storage**: 1Gi PVC for memory data persistence
- **Health Checks**: Liveness and readiness probes
- **Resource Limits**: CPU and memory limits for production use
- **ConfigMap**: Environment configuration management

### Deploy HTTP Mode

```bash
# Create namespace first (optional if using kubectl -n)
kubectl apply -f k8s-namespace.yaml

# Apply the HTTP mode deployment
kubectl apply -f k8s-deployment.yaml

# Check deployment status
kubectl get pods -l app=memory-mcp-server -n mcp
kubectl get svc memory-mcp-service -n mcp
kubectl get ingress memory-mcp-ingress -n mcp

# View logs
kubectl logs -l app=memory-mcp-server -n mcp -f
```

### Access the Service

1. **Internal Access** (within cluster):
   ```
   http://memory-mcp-service/sse
   ```

2. **External Access** (via ingress):
   - Update the host in the ingress: `memory-mcp.your-domain.com`
   - Access via: `http://memory-mcp.your-domain.com/sse`

### API Endpoints

- `GET /sse` - Establish SSE connection for MCP communication
- `POST /message?sessionId=xxx` - Send MCP requests

## STDIO Mode Deployment

The STDIO mode is suitable for integration with other services that communicate via standard input/output.

### Standard STDIO Mode (Internal Use Only)
- **Interactive Terminal**: stdin/tty enabled
- **Persistent Storage**: 1Gi PVC for memory data
- **Lower Resource Usage**: Optimized for STDIO operations
- **Limitation**: Only accessible within the cluster via kubectl exec

### STDIO Bridge Mode (External Access)
For external LLM clients to connect to STDIO mode, use the bridge deployment which provides:
- **HTTP-to-STDIO Bridge**: Converts HTTP requests to STDIO communication
- **Session Management**: Multiple concurrent client sessions
- **External Access**: Available via Ingress for external clients
- **Same MCP Protocol**: Full compatibility with MCP STDIO transport

### Deploy STDIO Mode

#### Standard STDIO Mode (Internal Only)
```bash
# Apply the STDIO mode deployment
kubectl apply -f k8s-stdio-deployment.yaml

# Check deployment status
kubectl get pods -l app=memory-mcp-stdio

# Connect to the pod for STDIO interaction
kubectl exec -it deployment/memory-mcp-stdio -- /bin/sh

# View logs
kubectl logs -l app=memory-mcp-stdio -f
```

#### STDIO Bridge Mode (External Access)
```bash
# Apply the STDIO bridge deployment
kubectl apply -f k8s-stdio-bridge.yaml

# Check deployment status
kubectl get pods -l app=memory-mcp-stdio-bridge
kubectl get svc memory-mcp-stdio-bridge-service
kubectl get ingress memory-mcp-stdio-bridge-ingress

# View logs
kubectl logs -l app=memory-mcp-stdio-bridge -f
```

### Interact with STDIO Mode

```bash
# Execute MCP commands via kubectl
echo '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}' | kubectl exec -i deployment/memory-mcp-stdio -- node dist/index.js
```

## Configuration

### Environment Variables

Both deployments support these environment variables:

- `MCP_TRANSPORT`: "stdio" or "http" (set automatically in deployments)
- `PORT`: HTTP server port (default: 3000, HTTP mode only)
- `MEMORY_FILE_PATH`: Path to memory storage file (default: "/data/memory.json")
- `NODE_ENV`: Node.js environment (set to "production")

### Customization

1. **Update Image Tag**:
   ```yaml
   image: nightlord851108/memory-mcp:v1.1.0
   ```

2. **Change Resources**:
   ```yaml
   resources:
     requests:
       memory: "128Mi"
       cpu: "100m"
     limits:
       memory: "512Mi"
       cpu: "500m"
   ```

3. **Configure Storage**:
   ```yaml
   resources:
     requests:
       storage: 5Gi
   ```

4. **Update Ingress Host**:
   ```yaml
   - host: memory-mcp.example.com
   ```

## Monitoring

### Check Service Health

```bash
# Check if pods are running
kubectl get pods

# Check service endpoints
kubectl get endpoints memory-mcp-service

# Test HTTP endpoint (from within cluster)
kubectl run test-pod --image=curlimages/curl --rm -it -- curl http://memory-mcp-service/sse
```

### View Logs

```bash
# HTTP mode logs
kubectl logs -l app=memory-mcp-server --tail=100 -f

# STDIO mode logs
kubectl logs -l app=memory-mcp-stdio --tail=100 -f
```

## Scaling

### HTTP Mode
```bash
kubectl scale deployment memory-mcp-server --replicas=3
```

### STDIO Mode
STDIO mode typically runs as a single instance due to its interactive nature.

## Cloudflare Tunnel Deployment

Cloudflare Tunnel provides secure external access to your MCP services without opening firewall ports.

### Features
- **Secure Access**: No inbound ports required
- **TLS Encryption**: Automatic HTTPS encryption
- **Multiple Services**: Route multiple hostnames to different services
- **Health Monitoring**: Built-in metrics and health checks
- **Access Control**: Integrate with Cloudflare Access

### Quick Setup

1. **Get Tunnel Token**: Create a tunnel in [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. **Configure Secret**: Update the tunnel token in the YAML file
3. **Update Domains**: Replace placeholder domains with your actual domains
4. **Deploy**: Apply the configuration

```bash
# Deploy Cloudflare Tunnel (token method)
kubectl apply -f k8s-cloudflare-tunnel.yaml

# Check tunnel status
kubectl get pods -n mcp -l app=cloudflare-tunnel
kubectl logs -n mcp -l app=cloudflare-tunnel -f

# Test external access
curl https://memory-mcp.yourdomain.com/health
```

### Configuration Options

**Token Method** (Recommended):
- Use `k8s-cloudflare-tunnel.yaml`
- Simple setup with tunnel token from dashboard
- Automatic DNS configuration

**Credentials Method**:  
- Use `k8s-cloudflare-tunnel-credentials.yaml`
- Uses traditional credentials.json approach
- More control over tunnel configuration

### Complete Setup Guide

See `cloudflare-tunnel-setup.md` for detailed setup instructions including:
- Step-by-step tunnel creation
- DNS configuration
- Advanced routing options
- Security best practices
- Troubleshooting guide

## Cleanup

```bash
# Remove HTTP mode deployment
kubectl delete -f k8s-deployment.yaml

# Remove STDIO mode deployment
kubectl delete -f k8s-stdio-deployment.yaml

# Remove Cloudflare Tunnel
kubectl delete -f k8s-cloudflare-tunnel.yaml

# Remove persistent volumes (optional)
kubectl delete pvc memory-mcp-pvc memory-mcp-stdio-pvc
```

## Troubleshooting

### Common Issues

1. **Pod Not Starting**:
   ```bash
   kubectl describe pod <pod-name>
   kubectl logs <pod-name>
   ```

2. **Service Not Accessible**:
   ```bash
   kubectl get svc
   kubectl describe svc memory-mcp-service
   ```

3. **Ingress Issues**:
   ```bash
   kubectl describe ingress memory-mcp-ingress
   kubectl get ingress
   ```

4. **Storage Issues**:
   ```bash
   kubectl get pvc
   kubectl describe pvc memory-mcp-pvc
   ```

### Health Check Failures

If health checks are failing, you can disable them temporarily:

```yaml
# Comment out or remove these sections
# livenessProbe:
#   httpGet:
#     path: /sse
#     port: 3000
# readinessProbe:
#   httpGet:
#     path: /sse
#     port: 3000
```