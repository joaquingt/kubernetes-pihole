# Pi-hole on Kubernetes

A complete Kubernetes deployment for Pi-hole DNS ad-blocker with persistent storage and ingress configuration.

## 📋 Prerequisites

### For Docker Desktop Kubernetes:
- Docker Desktop with Kubernetes enabled
- kubectl configured
- NGINX Ingress Controller

### For K3s (Raspberry Pi):
- K3s cluster running
- kubectl configured to connect to K3s
- Traefik Ingress Controller (included with K3s)

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Ingress       │    │   LoadBalancer  │    │   Deployment    │
│  pihole.local   │───▶│   Service       │───▶│   Pi-hole Pod   │
│  (Web UI)       │    │  (DNS + Web)    │    │   + Volumes     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## 📁 Project Structure

```
kubernetes-pihole/
├── pihole-namespace.yaml        # Namespace configuration
├── pihole-pvc.yaml             # Persistent Volume Claims (K3s compatible)
├── pihole-deployment.yaml      # Main Pi-hole deployment
├── pihole-svc.yaml            # ClusterIP service
├── pihole-svc-lb.yaml         # LoadBalancer service (optional)
├── pihole-ingress-traefik.yaml # Ingress for K3s (Traefik)
├── pihole-ingress-nginx.yaml  # Ingress for Docker Desktop (NGINX)
└── README.md                  # This file
```

## 🚀 Quick Start

### Docker Desktop Deployment

```bash
# Create namespace and persistent volumes
kubectl apply -f pihole-namespace.yaml
kubectl apply -f pihole-pvc.yaml

# Deploy Pi-hole
kubectl apply -f pihole-deployment.yaml

# Create service
kubectl apply -f pihole-svc.yaml

# Create ingress for Docker Desktop (NGINX)
kubectl apply -f pihole-ingress-nginx.yaml
```

### K3s (Raspberry Pi) Deployment

```bash
# Create namespace and persistent volumes (with explicit storage class)
kubectl apply -f pihole-namespace.yaml
kubectl apply -f pihole-pvc.yaml

# Deploy Pi-hole
kubectl apply -f pihole-deployment.yaml

# Create service
kubectl apply -f pihole-svc.yaml

# Create ingress (uses Traefik by default)
kubectl apply -f pihole-ingress-traefik.yaml
```

### 2. Access Pi-hole

#### Docker Desktop
- **Via Ingress**: `http://pihole.local` (add `127.0.0.1 pihole.local` to `/etc/hosts`)
- **Via Port-Forward**: `kubectl port-forward -n pihole service/pihole-service 8080:80`
- **Default Password**: `12345`

#### K3s (Raspberry Pi)
- **Via Ingress**: `http://pihole.local` (add `<RASPBERRY_PI_IP> pihole.local` to `/etc/hosts`)
- **Direct Access**: `http://<RASPBERRY_PI_IP>` (e.g., `http://192.168.1.73`)
- **Default Password**: `12345`

#### DNS Configuration
```bash
# Docker Desktop - Port-forward DNS to localhost
kubectl port-forward -n pihole service/pihole-service 53:53

# K3s - Use Raspberry Pi IP directly
# Set DNS in your devices to: <RASPBERRY_PI_IP> (e.g., 192.168.1.73)
```

## ⚙️ Configuration

### Environment Variables
Located in `pihole-deployment.yaml`:

```yaml
env:
  - name: WEBPASSWORD
    value: "12345"              # Change this!
  - name: PIHOLE_DNS_1
    value: "8.8.8.8"           # Primary upstream DNS
  - name: PIHOLE_DNS_2
    value: "8.8.4.4"           # Secondary upstream DNS
```

### Resource Limits
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "2Gi"
    cpu: "500m"
```

### Storage

#### Docker Desktop:
- Uses default storage class or omit `storageClassName`
- **Config Volume**: 1Gi (`/etc/pihole`)
- **DNS Config**: 2Gi (`/etc/dnsmasq.d`)

#### K3s:
- Uses `local-path` storage class (explicitly defined)
- **Config Volume**: 1Gi (`/etc/pihole`)
- **DNS Config**: 2Gi (`/etc/dnsmasq.d`)

## 🔧 Troubleshooting

### Common Issues

#### 1. SQLite Database Errors
```bash
# If you see "no such table: info" errors
kubectl exec -it <pod-name> -n pihole -- rm -f /etc/pihole/*.db*
kubectl delete pod <pod-name> -n pihole
```

#### 2. DNS Not Working
```bash
# Test DNS connectivity
dig @127.0.0.1 -p 53 google.com          # Docker Desktop
dig @<RASPBERRY_PI_IP> google.com         # K3s

# Check service status
kubectl get svc -n pihole
kubectl get pods -n pihole
```

#### 3. Web Interface 404
```bash
# Check ingress
kubectl get ingress -n pihole

# For Docker Desktop - Add to /etc/hosts
echo "127.0.0.1 pihole.local" | sudo tee -a /etc/hosts

# For K3s - Add to /etc/hosts  
echo "<RASPBERRY_PI_IP> pihole.local" | sudo tee -a /etc/hosts
```

#### 4. PVC Stuck in Pending (K3s)
```bash
# Check storage class
kubectl get storageclass

# Ensure PVCs use explicit storage class
# Update pihole-pvc.yaml to include: storageClassName: local-path
```

#### 5. Ingress Issues
```bash
# Docker Desktop - Check NGINX ingress
kubectl get pods -n ingress-nginx

# K3s - Check Traefik ingress
kubectl get pods -n kube-system | grep traefik
```

### Useful Commands

```bash
# Check pod logs
kubectl logs -f deployment/pihole-app -n pihole

# Get service details
kubectl describe svc pihole-service -n pihole

# Port-forward for local access
kubectl port-forward -n pihole service/pihole-service 8080:80 53:53

# Scale deployment
kubectl scale deployment pihole-app -n pihole --replicas=1
```

## 📊 Monitoring

### Health Checks
```bash
# Check pod status
kubectl get pods -n pihole -w

# View resource usage
kubectl top pods -n pihole

# Check events
kubectl get events -n pihole --sort-by='.lastTimestamp'
```

## 🔒 Security Notes

1. **Change Default Password**: Update `WEBPASSWORD` in deployment
2. **Network Policies**: Consider implementing network policies for DNS traffic
3. **TLS**: Enable HTTPS for web interface in production
4. **Backup**: Regular backup of PVC data recommended

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test the deployment
5. Submit a pull request

## 📝 License

This project is open source and available under the [MIT License](LICENSE).

## 🙏 Acknowledgments

- [Pi-hole](https://pi-hole.net/) - Network-wide ad blocking
- [Pi-hole Docker](https://github.com/pi-hole/docker-pi-hole) - Official Docker image

---

**Note**: This configuration supports both Docker Desktop Kubernetes and K3s deployments. For production use, consider additional security measures and proper DNS configuration.

## 📝 Deployment Differences

### Docker Desktop vs K3s Changes Made:

#### Storage:
- **Docker Desktop**: Can omit `storageClassName` (uses default)
- **K3s**: Requires explicit `storageClassName: local-path` in PVCs

#### Ingress:
- **Docker Desktop**: Uses `ingressClassName: nginx` (pihole-ingress-nginx.yaml)
- **K3s**: Uses `ingressClassName: traefik` (pihole-ingress-traefik.yaml)

#### Access:
- **Docker Desktop**: Access via localhost/port-forwarding
- **K3s**: Direct access via Raspberry Pi IP address