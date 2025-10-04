# FRP Tunnel Setup for SeaweedFS

This setup uses your VPS with a static IP as a stable tunnel endpoint for the SeaweedFS service.

## Architecture

```
Internet → VPS:9000 (frps server listener for S3 traffic)
              ↑ (control tunnel via port 7000)
         Cluster (frpc client) → SeaweedFS S3:8333
```

SeaweedFS's S3 API (port 8333 inside the cluster) is forwarded to TCP/9000 on your VPS for a single-hop TCP path—no HTTP ingress or additional proxies in the middle. Keep any other FRP listeners (for example the shared HTTPS tunnel on 443) open as before; the S3 gateway now has its own dedicated port for maximum throughput.

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

# Allow S3 ingress for SeaweedFS
sudo ufw allow 9000/tcp
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
- `frpc.yml` - FRP client that tunnels SeaweedFS directly to the filer S3 port (`seaweedfs-filer.services.svc.cluster.local:8333`) to avoid layered TCP proxies.

### 3. Configure SeaweedFS S3 identities

Under the Infisical path `/seaweedfs` add:
- `seaweedfs_s3_config.json` – full IAM payload (see example below).
- Any client-facing keys you want to reuse elsewhere (e.g., `S3_ACCESS_KEY`, `S3_SECRET_KEY`).

Example payload for `seaweedfs_s3_config.json`:

```json
{
  "identities": [
    {
      "name": "root",
      "accessKey": "YOUR_ACCESS_KEY",
      "secretKey": "YOUR_SECRET_KEY",
      "actions": ["Admin"]
    }
  ]
}
```

## DNS Setup

Point your SeaweedFS domain to your VPS IP:

```
s3.neurocollab.in → YOUR_VPS_IP
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

### Test SeaweedFS access:
```bash
# S3 API on port 9000 (forwarded directly from the cluster's filer gateway)
curl http://YOUR_VPS_IP:9000
# or via DNS once the record exists
curl http://s3.neurocollab.in:9000

# Or configure S3 client
aws s3 ls --endpoint-url http://s3.neurocollab.in:9000
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
- Check SeaweedFS filer service is running: `kubectl get svc -n services seaweedfs-filer`
- Verify local_ip in frpc config matches service name
- Check frps logs for proxy creation

### High latency
- Ensure VPS has good network connectivity
- Consider moving VPS closer to your cluster region
