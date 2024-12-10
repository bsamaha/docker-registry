# Local Docker Registry

This repository contains configuration for running a private Docker registry on WSL2 with K3s cluster support. The registry provides a local alternative to Docker Hub for storing and managing Docker images.

## Prerequisites

- Windows 10/11 with WSL2 enabled
- Docker Desktop installed and configured to work with WSL2
- OpenSSL for certificate generation
- K3s cluster (if using with Kubernetes)

## Important Security Notice

⚠️ **This setup uses self-signed certificates and is intended for development use only!**

- Self-signed certificates are not trusted by browsers or Docker clients by default
- This configuration is NOT suitable for production environments
- Do not expose this registry to the public internet
- Consider using proper CA-signed certificates for production use

## Directory Structure
.
├── config/
│ └── config.yml # Registry configuration
├── certs/ # SSL certificates (gitignored)
│ ├── domain.crt
│ └── domain.key
├── data/ # Registry storage (gitignored)
├── docker-compose.yml
└── README.md

## Initial Setup

### 1. Generate SSL Certificates

The registry requires SSL certificates for secure communication. Create them using:

```bash
# Create certs directory
mkdir -p certs

# Generate self-signed certificate with proper SANs
openssl req -x509 -newkey rsa:4096 -days 365 -nodes \
  -keyout certs/domain.key -out certs/domain.crt \
  -subj "/CN=registry.local" \
  -addext "subjectAltName=DNS:registry.local,IP:<REGISTRY_IP>,IP:<WSL_IP>"

# Create CA certificate
cp certs/domain.crt certs/ca.crt

# Set appropriate permissions
chmod 400 certs/domain.key
chmod 444 certs/domain.crt
chmod 444 certs/ca.crt
```

### 2. Configure Registry

Create config.yml:
```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /data
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
  secret: "mysecret123"
  tls:
    certificate: /certs/domain.crt
    key: /certs/domain.key
    minimumtls: tls1.2
maintenance:
  uploadpurging:
    enabled: true
    age: 168h
    interval: 24h
    dryrun: false
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

### 3. Create Docker Compose File

Create docker-compose.yml:
```yaml
version: '3'
services:
  registry:
    image: registry:latest
    ports:
      - "5001:5000"
    volumes:
      - ./data:/data
      - ./certs:/certs:ro
      - ./config.yml:/etc/docker/registry/config.yml:ro
    networks:
      - registry_net
    restart: always

networks:
  registry_net:
    driver: bridge
```

### 4. Start the Registry
```bash
docker-compose up -d
```

## K3s Cluster Setup

### 1. Configure K3s Nodes

Run these commands on each K3s node:

```bash
# From your WSL2 machine where the registry runs
# Define your K3s nodes - replace with your actual node IPs
NODES=("<NODE1_IP>" "<NODE2_IP>" "<NODE3_IP>")

# Loop through each node and set up the certificate
for NODE in "${NODES[@]}"; do
    echo "Setting up certificate on node $NODE..."
    
    # Copy the certificate (replace <USER> with your username)
    scp ~/docker-registry/certs/ca.crt <USER>@$NODE:/tmp/ca.crt
    
    # Run the setup commands on the remote node
    ssh <USER>@$NODE "
        sudo mkdir -p /usr/local/share/ca-certificates/docker-registry && \
        sudo mv /tmp/ca.crt /usr/local/share/ca-certificates/docker-registry/registry.crt && \
        sudo chmod 644 /usr/local/share/ca-certificates/docker-registry/registry.crt && \
        sudo update-ca-certificates
    "
done
```

### 2. Configure Registry in K3s

Create registries.yaml on each node:
```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "<REGISTRY_IP>:5001":
    endpoint:
      - "https://<REGISTRY_IP>:5001"
configs:
  "<REGISTRY_IP>:5001":
    tls:
      ca_file: /usr/local/share/ca-certificates/docker-registry/registry.crt
```

Deploy this configuration to all nodes:
```bash
# Create registries.yaml content
cat > /tmp/registries.yaml << 'EOF'
mirrors:
  "<REGISTRY_IP>:5001":
    endpoint:
      - "https://<REGISTRY_IP>:5001"
configs:
  "<REGISTRY_IP>:5001":
    tls:
      ca_file: /usr/local/share/ca-certificates/docker-registry/registry.crt
EOF

# Copy and apply to each node
for NODE in "${NODES[@]}"; do
    echo "Updating registries.yaml on node $NODE..."
    
    scp /tmp/registries.yaml <USER>@$NODE:/tmp/registries.yaml
    ssh <USER>@$NODE "
        sudo mv /tmp/registries.yaml /etc/rancher/k3s/registries.yaml && \
        sudo chmod 644 /etc/rancher/k3s/registries.yaml && \
        sudo systemctl restart k3s
    "
done
```

### 3. Verify Registry Access
```bash
# Test on each node
for NODE in "${NODES[@]}"; do
    echo "Testing node $NODE..."
    ssh <USER>@$NODE "curl --cacert /usr/local/share/ca-certificates/docker-registry/registry.crt https://<REGISTRY_IP>:5001/v2/_catalog"
    echo "----------------------------"
done
```

## Using the Registry

### Tag and Push Images
```bash
docker tag my-image:latest <REGISTRY_IP>:5001/my-image:latest
docker push <REGISTRY_IP>:5001/my-image:latest
```

### Pull Images
```bash
docker pull <REGISTRY_IP>:5001/my-image:latest
```

### Using in K3s Deployments
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: <REGISTRY_IP>:5001/my-app:latest
```

## Troubleshooting

### Certificate Issues
```bash
# Verify certificate installation
ls -la /usr/local/share/ca-certificates/docker-registry/
openssl x509 -in /usr/local/share/ca-certificates/docker-registry/registry.crt -text -noout
```

### Registry Configuration
```bash
# Verify registries.yaml
cat /etc/rancher/k3s/registries.yaml

# Check K3s logs
sudo journalctl -u k3s -f
```

### Network Connectivity
```bash
# Test basic connectivity
ping <REGISTRY_IP>
telnet <REGISTRY_IP> 5001
```

## Security Considerations

- ⚠️ **This setup uses self-signed certificates**
  - Not suitable for production environments
  - Certificates are not trusted by default
  - Must be manually added to trust stores
  - Should be regenerated periodically (every 365 days by default)

- **Access Control**
  - No authentication is implemented
  - Suitable for local development only
  - Anyone with network access can push/pull images
  - Consider implementing authentication for shared environments

- **Best Practices**
  - Keep certificates secure and regularly updated
  - Do not expose the registry to public networks
  - Use proper CA-signed certificates for production
  - Implement authentication if multiple users need access
  - Regular backups of registry data
  - Monitor disk space and certificate expiration

## Additional Resources

- [Docker Registry Documentation](https://docs.docker.com/registry/)
- [Registry Configuration Reference](https://docs.docker.com/registry/configuration/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [K3s Documentation](https://docs.k3s.io/)

## TODO: Using Real SSL Certificates

For production environments, replace self-signed certificates with real SSL certificates:

### 1. Obtain SSL Certificate

Options for obtaining real SSL certificates:
- Purchase from a trusted Certificate Authority (CA)
- Use Let's Encrypt for free certificates
- Use your organization's internal CA

### 2. Using Let's Encrypt (Recommended)

```bash
# Install certbot
sudo apt-get update
sudo apt-get install certbot

# Generate certificate (replace domain)
sudo certbot certonly --standalone -d registry.yourdomain.com

# Copy certificates to registry location
sudo cp /etc/letsencrypt/live/registry.yourdomain.com/fullchain.pem /path/to/registry/certs/domain.crt
sudo cp /etc/letsencrypt/live/registry.yourdomain.com/privkey.pem /path/to/registry/certs/domain.key

# Set permissions
sudo chmod 444 /path/to/registry/certs/domain.crt
sudo chmod 400 /path/to/registry/certs/domain.key
```

### 3. Certificate Auto-Renewal

Set up automatic renewal for Let's Encrypt certificates:

```bash
# Test renewal
sudo certbot renew --dry-run

# Add to crontab
sudo crontab -e

# Add this line to run twice daily
0 0,12 * * * certbot renew --quiet --post-hook "cp /etc/letsencrypt/live/registry.yourdomain.com/fullchain.pem /path/to/registry/certs/domain.crt && cp /etc/letsencrypt/live/registry.yourdomain.com/privkey.pem /path/to/registry/certs/domain.key && docker restart registry"
```

### 4. Update Registry Configuration

Update config.yml to use the new certificates:
```yaml
http:
  addr: :5000
  tls:
    certificate: /certs/domain.crt
    key: /certs/domain.key
    minimumtls: tls1.2
    # Remove insecure registry settings once real certs are in place
```

### 5. DNS Configuration

1. Set up proper DNS records for your registry domain
2. Configure firewall rules for ports 80 (HTTP) and 443 (HTTPS)
3. Ensure domain is accessible from all nodes

### 6. Security Considerations

When using real certificates:
- Keep private keys secure
- Monitor certificate expiration
- Use strong SSL/TLS configuration
- Implement proper access controls
- Regular security audits
- Consider using a Web Application Firewall (WAF)

### 7. Additional Recommendations

- Use separate staging/production registries
- Implement authentication (OAuth or LDAP)
- Set up monitoring for certificate expiration
- Regular backups of registry data
- Consider using a CDN for better performance
- Implement rate limiting

## Certificate Maintenance

### Checking and Renewing Self-Signed Certificates

1. **Check Current Certificate Expiration**
```bash
# Check certificate expiration date
openssl x509 -in certs/domain.crt -text -noout | grep "Not After"
```

2. **Backup Existing Certificates**
```bash
# Create backup directory
mkdir -p certs_backup
cp certs/domain.* certs_backup/
```

3. **Stop the Registry**
```bash
docker-compose down
```

4. **Generate New Certificates**
```bash
# Generate new self-signed certificate
openssl req -x509 -newkey rsa:4096 -days 365 -nodes \
  -keyout certs/domain.key -out certs/domain.crt \
  -subj "/CN=registry.local" \
  -addext "subjectAltName=DNS:registry.local,IP:<REGISTRY_IP>,IP:<WSL_IP>"

# Create new CA certificate
cp certs/domain.crt certs/ca.crt

# Set appropriate permissions
chmod 400 certs/domain.key
chmod 444 certs/domain.crt
chmod 444 certs/ca.crt
```

5. **Update K3s Nodes**
```bash
# Define your K3s nodes
NODES=("<NODE1_IP>" "<NODE2_IP>" "<NODE3_IP>")

for NODE in "${NODES[@]}"; do
    echo "Updating certificate on node $NODE..."
    
    # Copy the new certificate
    scp certs/ca.crt <USER>@$NODE:/tmp/ca.crt
    
    # Update on the remote node
    ssh <USER>@$NODE "
        sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/docker-registry/registry.crt && \
        sudo chmod 644 /usr/local/share/ca-certificates/docker-registry/registry.crt && \
        sudo update-ca-certificates && \
        sudo systemctl restart k3s
    "
done
```

6. **Restart Registry**
```bash
docker-compose up -d
```

7. **Verify New Certificate**
```bash
# Check new expiration date
openssl x509 -in certs/domain.crt -text -noout | grep "Not After"

# Test registry access
curl --cacert certs/ca.crt https://<REGISTRY_IP>:5001/v2/_catalog
```

8. **Verify K3s Node Access**
```bash
# Test on each node
for NODE in "${NODES[@]}"; do
    echo "Testing node $NODE..."
    ssh <USER>@$NODE "curl --cacert /usr/local/share/ca-certificates/docker-registry/registry.crt https://<REGISTRY_IP>:5001/v2/_catalog"
    echo "----------------------------"
done
```

9. **Cleanup (Optional)**
```bash
# If everything works, you can remove the backup
# rm -rf certs_backup

# Or keep it for a while just in case
mv certs_backup certs_backup_$(date +%Y%m%d)
```

### Troubleshooting Certificate Renewal

If you encounter issues after renewal:

1. **Registry Issues**
```bash
# Check registry logs
docker-compose logs

# Verify certificate permissions
ls -la certs/

# Test registry directly
curl -v --cacert certs/ca.crt https://<REGISTRY_IP>:5001/v2/_catalog
```

2. **K3s Node Issues**
```bash
# Check certificate on nodes
ssh <USER>@<NODE_IP> "sudo ls -la /usr/local/share/ca-certificates/docker-registry/"

# Verify K3s can access registry
ssh <USER>@<NODE_IP> "sudo k3s crictl pull <REGISTRY_IP>:5001/some-test-image:latest"

# Check K3s logs
ssh <USER>@<NODE_IP> "sudo journalctl -u k3s -f"
```

3. **Recovery**
```bash
# If needed, restore old certificates
cp certs_backup/* certs/
docker-compose up -d

# Restore on K3s nodes
for NODE in "${NODES[@]}"; do
    scp certs_backup/ca.crt <USER>@$NODE:/tmp/ca.crt
    ssh <USER>@$NODE "
        sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/docker-registry/registry.crt && \
        sudo update-ca-certificates && \
        sudo systemctl restart k3s
    "
done
```

Remember to schedule regular certificate renewals before expiration (recommended: renew 30 days before expiry).
