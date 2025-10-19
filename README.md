# Pi-hole on Kubernetes

A complete Kubernetes deployment for Pi-hole DNS ad-blocker with persistent storage and ingress configuration.

## 📋 Prerequisites

- Kubernetes cluster (tested with Docker Desktop)
- kubectl configured
- NGINX Ingress Controller (for web interface)

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
├── pihole-namespace.yaml     # Namespace configuration
├── pihole-pvc.yaml          # Persistent Volume Claims
├── pihole-deployment.yaml   # Main Pi-hole deployment
├── pihole-svc.yaml         # ClusterIP service
├── pihole-svc lb.yaml      # LoadBalancer service
├── pihole-ingress.yaml     # Ingress for web UI
└── README.md               # This file
```

## 🚀 Quick Start

### 1. Deploy Pi-hole

```bash
# Create namespace and persistent volumes
kubectl apply -f pihole-namespace.yaml
kubectl apply -f pihole-pvc.yaml

# Deploy Pi-hole
kubectl apply -f pihole-deployment.yaml

# Create service (choose one)
kubectl apply -f pihole-svc.yaml          # ClusterIP (recommended)
# OR
kubectl apply -f pihole-svc\ lb.yaml      # LoadBalancer

# Create ingress for web interface
kubectl apply -f pihole-ingress.yaml
```

### 2. Access Pi-hole

#### Web Interface
- **Via Ingress**: `http://pihole.local` (add to `/etc/hosts`)
- **Via Port-Forward**: `kubectl port-forward -n pihole service/pihole-service 8080:80`
- **Default Password**: `12345`

#### DNS Configuration
```bash
# Port-forward DNS to localhost
kubectl port-forward -n pihole service/pihole-service 53:53

# Then set DNS in your system to: 127.0.0.1
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
dig @127.0.0.1 -p 53 google.com

# Check service status
kubectl get svc -n pihole
kubectl get pods -n pihole
```

#### 3. Web Interface 404
```bash
# Check ingress
kubectl get ingress -n pihole

# Add to /etc/hosts if using ingress
echo "127.0.0.1 pihole.local" | sudo tee -a /etc/hosts
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

**Note**: This configuration is optimized for local development with Docker Desktop Kubernetes. For production use, consider additional security measures and proper DNS configuration.