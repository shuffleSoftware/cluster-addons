# ingress-nginx Configuration for External Access

This guide explains how to configure ingress-nginx to allow incoming requests from the internet.

## Overview

The ingress-nginx controller is configured with a LoadBalancer service type which automatically creates a Public IP in Azure. To allow incoming requests, you need to:

1. ✅ **Network Security Groups** - Already configured in your AKS infrastructure
2. ✅ **LoadBalancer Service** - Already configured 
3. ⚠️ **Controller Configuration** - Needs optimization for external traffic
4. ❌ **DNS Configuration** - Point your domains to the Public IP
5. ❌ **Ingress Resources** - Define routing rules for your applications

## Step-by-Step Configuration

### 1. Deploy/Update ingress-nginx

```bash
# Navigate to the ingress-nginx directory
cd cluster-addons/cluster-addons/ingress-nginx/dev

# Deploy or update the ingress-nginx controller
helm upgrade --install ingress-nginx . -n ingress-nginx --create-namespace
```

### 2. Get the Public IP Address

```bash
# Get the external IP address assigned by the LoadBalancer
kubectl get service ingress-nginx-controller -n ingress-nginx

# Output will show something like:
# NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
# ingress-nginx-controller   LoadBalancer   10.0.123.456   20.22.24.26     80:31234/TCP,443:32567/TCP
```

### 3. Configure DNS

Point your domain names to the EXTERNAL-IP from step 2:

```dns
# Example DNS A records
myapp.example.com     A    20.22.24.26
api.example.com       A    20.22.24.26
*.example.com         A    20.22.24.26  # Wildcard for subdomains
```

### 4. Create Ingress Resources

Create ingress resources for your applications (see `example-ingress.yaml` for reference):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: my-app-namespace
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

### 5. Test External Access

```bash
# Test HTTP access
curl -H "Host: myapp.example.com" http://EXTERNAL-IP/

# Test with actual domain (after DNS propagation)
curl http://myapp.example.com/
```

## Security Considerations

### 1. Restrict Source IPs (Optional)
If you want to restrict access to specific IP ranges, uncomment and modify the `loadBalancerSourceRanges` in `values.yaml`:

```yaml
service:
  loadBalancerSourceRanges:
    - "203.0.113.0/24"  # Your office IP range
    - "198.51.100.0/24" # Another allowed IP range
```

### 2. Enable HTTPS/TLS

```yaml
# In your ingress resource
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret
```

Create TLS secret:
```bash
kubectl create secret tls myapp-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n my-app-namespace
```

### 3. Enable cert-manager for Automatic SSL
Consider installing cert-manager for automatic SSL certificate management:

```bash
# Add cert-manager repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

## Troubleshooting

### 1. Check ingress-nginx Status
```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### 2. Check Service and Endpoints
```bash
kubectl get svc -n ingress-nginx
kubectl get endpoints -n ingress-nginx
```

### 3. Test Internal Connectivity
```bash
# From within the cluster
kubectl run test-pod --rm -i --tty --image=curlimages/curl -- sh
# Then inside the pod:
curl http://ingress-nginx-controller.ingress-nginx.svc.cluster.local
```

### 4. Check Ingress Resources
```bash
kubectl get ingress --all-namespaces
kubectl describe ingress my-app-ingress -n my-app-namespace
```

## Configuration Details

### External Traffic Policy: Local
- **Benefit**: Preserves source IP addresses
- **Trade-off**: May cause uneven load distribution
- **Alternative**: Set to `Cluster` if you don't need source IP preservation

### Use-Forwarded-Headers
- Enables proper handling of X-Forwarded-For headers
- Important for applications that need to know the original client IP

### Default Ingress Class
- Makes this ingress controller the default for new ingress resources
- Remove `default: true` if you have multiple ingress controllers

## Next Steps

1. Update your applications to use proper service definitions
2. Create ingress resources for each application
3. Configure DNS to point to your LoadBalancer IP
4. Set up SSL/TLS certificates
5. Configure monitoring and logging
6. Implement security policies (NetworkPolicies, etc.)

## Monitoring

Consider setting up monitoring for your ingress controller:
- Enable Prometheus metrics
- Set up Grafana dashboards
- Configure alerting for ingress failures 