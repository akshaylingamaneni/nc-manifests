# FRP Tunnel Setup for MinIO

This setup uses your VPS with a static IP as a stable tunnel endpoint for MinIO service.

## Architecture

```
Internet → VPS:443 (frps server)
              ↑ (tunnel via port 7000)
         Cluster (frpc client) → MinIO:9000
```

MinIO's S3 API (port 9000) is tunneled to port 443 on your VPS.

## VPS Setup (One-time, Manual)

### 1. Install frps on your VPS

```bash
# Download frp (check for latest version)
wget https://github.com/fatedier/frp/releases/download/v0.61.0/frp_0.61.0_linux_amd64.tar.gz
tar -xzf frp_0.61.0_linux_amd64.tar.gz
cd frp_0.61.0_linux_amd64
sudo cp frps /usr/local/bin/
```

### 2. Create frps config

```bash
sudo mkdir -p /etc/frp
sudo nano /etc/frp/frps.toml
```

Add this content:

```toml
bindPort = 7000

auth.method = "token"
auth.token = "YOUR_LONG_RANDOM_TOKEN"
```

### 3. Create systemd service

```bash
sudo nano /etc/systemd/system/frps.service
```

Add:

```ini
[Unit]
Description=FRP Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/frps -c /etc/frp/frps.toml
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

### 4. Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
sudo systemctl status frps
```

### 5. Configure firewall

```bash
# Allow FRP control port
sudo ufw allow 7000/tcp

# Allow HTTPS port for MinIO
sudo ufw allow 443/tcp
```

## Cluster Setup (GitOps)

### 1. Add FRP secrets to Infisical

In your Infisical dashboard:
- Project: `uio-bd-ns`
- Environment: `prod`
- Path: `/frpc`
- Add secrets:
  - `VPS_IP` = Your VPS IP address (e.g., `203.0.113.42`)
  - `FRP_TOKEN` = `YOUR_LONG_RANDOM_TOKEN` (same as VPS)

### 2. Deploy

The GitOps will automatically deploy:
- `frpc-secrets-infisical.yml` - Pulls secrets from Infisical
- `frpc.yml` - FRP client that tunnels MinIO

## DNS Setup

Point your MinIO domain to your VPS IP:

```
minio.neurocollab.in → YOUR_VPS_IP
```

## Testing

### Check frps logs on VPS:
```bash
sudo journalctl -u frps -f
```

### Check frpc logs in cluster:
```bash
kubectl logs -n services -l app=frpc -f
```

### Test MinIO access:
```bash
# S3 API on port 443
curl http://YOUR_VPS_IP:443

# Or configure S3 client
aws s3 ls --endpoint-url http://YOUR_VPS_IP:443
```

## Benefits

✅ **Stable IP**: No need to update DNS when cluster restarts  
✅ **Simple**: One VPS, minimal config  
✅ **Secure**: Token-based auth, encrypted tunnel  
✅ **Cost-effective**: Uses existing VPS  
✅ **GitOps-friendly**: Only client side is managed

## Troubleshooting

### Connection refused
- Check frps is running: `systemctl status frps`
- Check firewall rules: `sudo ufw status`
- Verify token matches on both sides

### Tunnel connects but no traffic
- Check MinIO service is running: `kubectl get svc -n services minio`
- Verify local_ip in frpc config matches service name
- Check frps logs for proxy creation

### High latency
- Ensure VPS has good network connectivity
- Consider moving VPS closer to your cluster region

