# ArgoCD Setup and Configuration Guide

## Prerequisites
- Kubernetes cluster running (EKS/minikube/etc.)
- kubectl configured with appropriate context
- ArgoCD installed in the cluster

## 1. Configure ArgoCD Server in Insecure Mode

### Get and Edit ConfigMap
```bash
# List configmaps in argocd namespace
kubectl get cm -n argocd

# Edit the argocd-cmd-params-cm configmap
kubectl edit cm argocd-cmd-params-cm -n argocd
```

### Add the following data to the configmap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"
```

### Restart ArgoCD server to apply changes:
```bash
kubectl rollout restart deployment argocd-server -n argocd
```

## 2. Expose ArgoCD Server Service

### Change Service Type from ClusterIP to NodePort
```bash
# Edit the argocd-server service
kubectl edit svc argocd-server -n argocd
```

### Change the service type:
```yaml
spec:
  type: NodePort  # Change from ClusterIP to NodePort
```

### Get the NodePort and access details:
```bash
# Get service details including NodePort
kubectl get svc argocd-server -n argocd

# Get node external IPs
kubectl get nodes -o wide

# Get specific HTTPS NodePort
kubectl get svc argocd-server -n argocd -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
```

## 3. Configure Security Groups (for AWS EKS)

### Allow inbound traffic on NodePort:
- Navigate to AWS EC2 Console → Security Groups
- Find the security group attached to your worker nodes
- Add inbound rules for the NodePort (typically 30000-32767 range)
- Source: 0.0.0.0/0 (or restrict to your IP range)

## 4. Access ArgoCD UI

### Get Initial Admin Password
```bash
# Retrieve and decode the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### Login Credentials:
- **Username**: `admin`
- **Password**: (decoded password from above command)
- **URL**: `https://<node-public-ip>:<nodeport>` or `http://<node-public-ip>:<nodeport>`

## 5. Install ArgoCD CLI

### Download and install ArgoCD CLI:
```bash
# For Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# For macOS
brew install argocd

# Verify installation
argocd version
```

## 6. Add Clusters via CLI

### Login to ArgoCD via CLI
```bash
# Login to ArgoCD server
argocd login <argocd-server-ip>:<port>

# Use admin credentials when prompted
```

### Add Additional Clusters
```bash
# List available contexts
kubectl config get-contexts

# Add cluster to ArgoCD (must be done via CLI for security)
argocd cluster add <context-name>

# Verify clusters were added
argocd cluster list
```

### Example:
```bash
# Add a cluster context named 'spoke-cluster'
argocd cluster add spoke-cluster

# List all managed clusters
argocd cluster list
```

## 7. Create Applications

### Via ArgoCD UI:
1. Navigate to ArgoCD web interface
2. Click **"+ NEW APP"**
3. Fill in application details:
   - **Application Name**: your-app-name
   - **Project**: default
   - **Sync Policy**: Manual/Automatic
   - **Repository URL**: your-git-repo-url
   - **Path**: path-to-kubernetes-manifests
   - **Destination Cluster**: target-cluster-url
   - **Namespace**: target-namespace

### Via CLI (Alternative):
```bash
# Create application via CLI
argocd app create <app-name> \
  --repo <git-repo-url> \
  --path <path-to-manifests> \
  --dest-server <cluster-server-url> \
  --dest-namespace <namespace>

# Sync application
argocd app sync <app-name>
```

## 8. Verification

### Verify Setup:
```bash
# Check ArgoCD pods status
kubectl get pods -n argocd

# Check service exposure
kubectl get svc -n argocd

# Check managed clusters
argocd cluster list

# Check applications
argocd app list
```

## Troubleshooting

### Common Issues:
- **UI not accessible**: Check security groups and NodePort configuration
- **Login issues**: Verify admin password and ensure server is in insecure mode
- **Cluster addition fails**: Ensure kubectl context is properly configured
- **Application sync issues**: Check repository access and manifest validity

### Useful Commands:
```bash
# Reset admin password
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa","admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'

# View ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server -f

# Port forward for local access (alternative to NodePort)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Security Considerations

- **Insecure Mode**: Only use in development/testing environments
- **Network Access**: Restrict security group rules to necessary IP ranges
- **Cluster Access**: ArgoCD will have admin access to added clusters
- **Repository Access**: Ensure proper authentication for private repositories