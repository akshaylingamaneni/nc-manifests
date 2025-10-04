┌─────────────────────────────────────────────────────────────────┐
│ 1. YOUR LOCAL MACHINE                                           │
│    aws s3 cp file.txt s3://test1/                              │
│    ↓ AWS CLI creates HTTPS request                             │
│    ↓ Encrypts with TLS                                         │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
                    Your Home Router
                             ↓
                    Your ISP (370 Mbps)
                             ↓
                        INTERNET
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. YOUR VPS (Frankfurt - 186 Mbps)                             │
│    Receives: https://s3.neurocollab.in:443                     │
│    ↓                                                            │
│    frps (FRP Server) listening on :443                         │
│    ↓ Decrypts FRP tunnel (currently 1 TCP connection)          │
│    ↓ Sends data BACK through internet to your cluster          │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
                        INTERNET (again!)
                             ↓
                    Your ISP (370 Mbps download)
                             ↓
                    Your Home Router
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. YOUR HOME CLUSTER                                            │
│    frpc pod receives from VPS                                   │
│    ↓ Routes to nginx-ingress                                   │
│    ↓ nginx-ingress (TLS termination)                           │
│    ↓ Decrypts HTTPS, checks certificate                        │
│    ↓ Routes based on Host: s3.neurocollab.in                   │
│    ↓                                                            │
│    seaweedfs-filer pod (port 8333 - S3 API)                   │
│    ↓ S3 signature validation (IAM)                             │
│    ↓ Determines which volume server to use                     │
│    ↓                                                            │
│    seaweedfs-volume-0/1/2 pod                                  │
│    ↓ Writes to disk: /data0 or /data1                         │
│    ↓ PVC: seaweedfs-volume-X (70Gi each)                      │
│    ↓ Physical storage: microk8s-hostpath                       │
│    └─→ Actual file stored on your server's disk               │
└─────────────────────────────────────────────────────────────────┘