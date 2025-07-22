# HAProxy Blue-Green Deployment

This project provides an Ansible-based solution for deploying HAProxy as a blue-green load balancer that can route traffic between different cluster environments based on subdomain rewriting.

## Features

- **Blue-Green Routing**: Automatically routes traffic from generic subdomains (e.g., `dev.example.com`) to specific blue or green environments (e.g., `dev-blue.example.com` or `dev-green.example.com`)
- **Multiple Deployment Options**: Support for both Podman (with Quadlet) and Kubernetes deployments
- **SSL/TLS Support**: Configurable SSL termination with automatic HTTP to HTTPS redirection
- **Health Monitoring**: Built-in HAProxy stats page and health checks
- **Configurable Routing**: Easy configuration of cluster mappings and backend servers
- **Container Security**: Implements security best practices with minimal privileges

## Architecture

The system uses HAProxy to:

1. **Listen** on standard HTTP (80) and HTTPS (443) ports
2. **Inspect** the incoming `Host` header
3. **Rewrite** generic subdomains to environment-specific subdomains
4. **Route** traffic to the appropriate blue or green backend servers

### Example Routing

| Source Request | Routed To (Default: Blue) | Alternative (Green) |
|---------------|---------------------------|-------------------|
| `dev.example.com` | `dev-blue.example.com` | `dev-green.example.com` |
| `staging.example.com` | `staging-blue.example.com` | `staging-green.example.com` |
| `prod.example.com` | `prod-blue.example.com` | `prod-green.example.com` |

## Prerequisites

### For Ansible Deployment

- Ansible 2.9+
- Python 3.6+
- For Podman deployment: Podman 4.0+ with Quadlet support
- For Kubernetes deployment: `kubernetes.core` Ansible collection

### For Manual Deployment

- Podman 4.0+ or Kubernetes cluster access
- Basic understanding of container orchestration

## Quick Start

### 1. Ansible Deployment

```bash
# Clone the repository
git clone <repository-url>
cd ocp4-cluster-workload-migration

# Configure your environment
cp vars/haproxy-blue-green.yml vars/haproxy-blue-green.local.yml
# Edit vars/haproxy-blue-green.local.yml with your specific settings

# Deploy with Podman
ansible-playbook configure-haproxy-blue-green-deployment.yml \
  -e deployment_method=podman \
  -e @vars/haproxy-blue-green.local.yml

# Deploy with Kubernetes
ansible-playbook configure-haproxy-blue-green-deployment.yml \
  -e deployment_method=kubernetes \
  -e @vars/haproxy-blue-green.local.yml
```

### 2. Manual Podman Deployment

```bash
# Create required directories
sudo mkdir -p /opt/haproxy/{config,ssl}
sudo chown -R 1000:1000 /opt/haproxy

# Copy configuration files
sudo cp examples/haproxy.cfg /opt/haproxy/config/
sudo cp examples/domain_map.txt /opt/haproxy/config/

# Copy Quadlet files
sudo cp manifests/podman/*.{container,network} /etc/containers/systemd/

# Start services
sudo systemctl daemon-reload
sudo systemctl start haproxy-network.service
sudo systemctl start haproxy-blue-green.service
sudo systemctl enable haproxy-blue-green.service
```

### 3. Manual Kubernetes Deployment

```bash
# Apply all manifests
kubectl apply -f manifests/kubernetes/

# Check deployment status
kubectl get pods -n haproxy-system
kubectl get svc -n haproxy-system
```

## Configuration

### Basic Configuration

Edit `vars/haproxy-blue-green.yml` to customize your deployment:

```yaml
# Deployment method
deployment_method: "podman"  # or "kubernetes"

# HAProxy container configuration
haproxy:
  image: "docker.io/haproxy:2.8"
  http_port: 80
  https_port: 443
  stats_port: 8404

# Blue-Green routing configuration
blue_green_routing:
  default_environment: "blue"  # or "green"
  
  cluster_mappings:
    - source_subdomain: "dev"
      blue_subdomain: "dev-blue"
      green_subdomain: "dev-green"
```

### Backend Server Configuration

Configure your backend servers in the `backends` section:

```yaml
blue_green_routing:
  backends:
    dev-blue:
      servers:
        - name: "dev-blue-1"
          address: "dev-blue.example.com:80"
          check: "check"
        - name: "dev-blue-2"
          address: "dev-blue-backup.example.com:80"
          check: "check backup"
```

### SSL Configuration

Enable SSL/TLS termination:

```yaml
ssl_config:
  enabled: true
  cert_path: "/etc/ssl/certs/haproxy.crt"
  key_path: "/etc/ssl/private/haproxy.key"
  redirect_http_to_https: true
```

## Switching Between Blue and Green

### Method 1: Configuration Change

1. Edit your configuration file
2. Change `default_environment` from "blue" to "green" (or vice versa)
3. Redeploy using Ansible

### Method 2: Dynamic Switching (Advanced)

For runtime switching without downtime, you can:

1. Update the domain mapping file directly
2. Reload HAProxy configuration
3. Use HAProxy's runtime API for zero-downtime switches

## Monitoring and Troubleshooting

### HAProxy Stats

Access the HAProxy statistics page at:
- Podman: `http://localhost:8404/stats`
- Kubernetes: `http://<service-ip>:8404/stats`

Default credentials: `admin` / `changeme`

### Health Checks

The system includes several health checks:

- **Container Health**: HAProxy configuration validation
- **Port Connectivity**: Verify all ports are accessible
- **Backend Health**: Monitor backend server status
- **Routing Tests**: Validate blue-green routing logic

### Logs

#### Podman Logs
```bash
# Container logs
sudo podman logs haproxy-blue-green

# Systemd service logs
sudo journalctl -u haproxy-blue-green.service -f
```

#### Kubernetes Logs
```bash
# Pod logs
kubectl logs -n haproxy-system deployment/haproxy-blue-green

# Service events
kubectl describe svc -n haproxy-system haproxy-blue-green
```

### Common Issues

1. **Port Already in Use**
   - Check if another service is using ports 80, 443, or 8404
   - Use `netstat -tlnp` or `ss -tlnp` to identify conflicts

2. **Backend Connection Failures**
   - Verify backend server addresses are reachable
   - Check firewall rules and network connectivity
   - Review HAProxy logs for connection errors

3. **SSL Certificate Issues**
   - Ensure certificate files exist and are readable
   - Verify certificate format (PEM)
   - Check certificate expiration dates

## Security Considerations

- **Container Security**: Runs as non-root user with minimal capabilities
- **SSL/TLS**: Uses modern cipher suites and TLS 1.2+
- **Network Isolation**: Podman deployment uses dedicated network
- **Secret Management**: SSL certificates should be managed securely
- **Access Control**: Stats page should be password-protected

## File Structure

```
├── configure-haproxy-blue-green-deployment.yml  # Main playbook
├── vars/haproxy-blue-green.yml                  # Configuration variables
├── tasks/
│   ├── configure-haproxy-blue-green.yml         # HAProxy configuration
│   ├── deploy-haproxy-podman.yml                # Podman deployment
│   ├── deploy-haproxy-kubernetes.yml            # Kubernetes deployment
│   └── validate-haproxy-deployment.yml          # Validation tasks
├── templates/
│   ├── haproxy-blue-green.cfg.j2               # HAProxy config template
│   ├── haproxy-domain-map.txt.j2               # Domain mapping template
│   ├── haproxy-quadlet.container.j2            # Quadlet container template
│   └── haproxy-quadlet.network.j2              # Quadlet network template
├── handlers/haproxy-blue-green.yml              # Service handlers
├── manifests/
│   ├── kubernetes/                              # Kubernetes manifests
│   └── podman/                                  # Podman Quadlet examples
└── docs/HAPROXY-BLUE-GREEN-README.md           # This documentation
```

## Contributing

1. Test changes in a development environment
2. Update documentation for any configuration changes
3. Ensure both Podman and Kubernetes deployments work
4. Add appropriate validation tests

## License

This project is part of the OCP4 Cluster Workload Migration toolkit and follows the same licensing terms. 