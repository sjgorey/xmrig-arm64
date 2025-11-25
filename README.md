# XMRig ARM64 Docker Image

![Build Status](https://github.com/sjgorey/xmrig-arm64/actions/workflows/build-and-push.yml/badge.svg)

A Docker image for XMRig cryptocurrency miner compiled for ARM64 architecture, specifically optimized for ODROID and other ARM64 devices running on Kubernetes.

## Overview

This repository builds a minimal, secure XMRig Docker image from source for ARM64 platforms. The image is designed to run on Kubernetes as a DaemonSet across ARM64 nodes.

## Features

- **ARM64 Native**: Compiled specifically for ARM64 architecture
- **From Source**: Built directly from official XMRig repository
- **Minimal Size**: Multi-stage Alpine-based build for small footprint
- **Hardware Optimized**: Includes HWLOC support for hardware topology detection
- **Huge Pages**: Support for huge pages for improved performance
- **Automated Builds**: GitHub Actions automatically build and publish on every commit

## Image Details

- **Base**: Alpine Linux (ARM64)
- **XMRig Version**: v6.21.0 (configurable)
- **Architecture**: linux/arm64
- **Registry**: Docker Hub (`sjgorey/xmrig-arm64`)

## Quick Start

### Pull the Image

```bash
docker pull sjgorey/xmrig-arm64:latest
```

### Run Manually

```bash
docker run -d \
  --name xmrig \
  --cap-add=SYS_ADMIN \
  -v /dev/hugepages:/dev/hugepages \
  sjgorey/xmrig-arm64:latest \
  --url=p2pool.io:3333 \
  --user=YOUR_WALLET_ADDRESS \
  --pass=WORKER_NAME \
  --tls \
  --donate-level=1
```

### Kubernetes Deployment

See the [home-argocd-app-config](https://github.com/sjgorey/home-argocd-app-config) repository under `infrastructure/mining/` for complete Kubernetes manifests.

Example DaemonSet usage:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: xmrig
  namespace: mining
spec:
  selector:
    matchLabels:
      app: xmrig
  template:
    metadata:
      labels:
        app: xmrig
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      containers:
      - name: xmrig
        image: sjgorey/xmrig-arm64:latest
        env:
        - name: POOL_URL
          value: "p2pool.io:3333"
        - name: POOL_USER
          valueFrom:
            secretKeyRef:
              name: xmrig-secrets
              key: wallet
        - name: POOL_PASS
          valueFrom:
            secretKeyRef:
              name: xmrig-secrets
              key: worker-name
        securityContext:
          capabilities:
            add: ["SYS_ADMIN"]
        volumeMounts:
        - name: hugepages
          mountPath: /dev/hugepages
      volumes:
      - name: hugepages
        hostPath:
          path: /dev/hugepages
```

## Configuration

XMRig accepts standard command-line arguments. Common options:

| Option | Description | Example |
|--------|-------------|---------|
| `--url` | Pool URL | `p2pool.io:3333` |
| `--user` | Wallet address | Your Monero wallet |
| `--pass` | Worker name/password | `worker01` |
| `--tls` | Enable TLS | Flag (no value) |
| `--donate-level` | Donation percentage | `1` (1%) |
| `--threads` | Number of mining threads | `4` |
| `--cpu-priority` | CPU priority (0-5) | `5` |

For all options, see the [XMRig documentation](https://xmrig.com/docs/miner/command-line-options).

## Building Locally

### Prerequisites
- Docker with BuildX support
- QEMU for ARM64 emulation (if building on AMD64)

### Build Commands

```bash
# Clone the repository
git clone https://github.com/sjgorey/xmrig-arm64.git
cd xmrig-arm64

# Build for ARM64
docker buildx build \
  --platform linux/arm64 \
  -f Dockerfile.xmrig \
  -t sjgorey/xmrig-arm64:latest \
  .
```

### Build on ARM64 Device

If you're building directly on an ARM64 device (like ODROID):

```bash
docker build -f Dockerfile.xmrig -t sjgorey/xmrig-arm64:latest .
```

## GitHub Actions

This repository uses GitHub Actions for automated builds. See [.github/ACTIONS.md](.github/ACTIONS.md) for setup instructions.

### Automated Builds
- **On push to main**: Builds and pushes with `latest` tag
- **On version tags**: Builds and pushes with version tags
- **On pull requests**: Test build only (no push)

### Required Secrets
- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub access token

## Performance Optimization

### Host System Requirements

For optimal performance, configure huge pages on your ARM64 nodes:

```bash
# Check current huge pages
cat /proc/meminfo | grep Huge

# Set huge pages (add to /etc/sysctl.conf)
vm.nr_hugepages = 128

# Apply immediately
sudo sysctl -w vm.nr_hugepages=128
```

### Resource Limits

Recommended Kubernetes resource limits:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"
  limits:
    cpu: "4"
    memory: "4Gi"
```

## Updating XMRig Version

To update to a newer version of XMRig:

1. Edit `Dockerfile.xmrig`
2. Change the git clone branch: `--branch v6.21.0` to the new version
3. Commit and push
4. Tag the release:
   ```bash
   git tag v6.22.0
   git push origin v6.22.0
   ```

GitHub Actions will automatically build the new version.

## Security Considerations

- **Capability Requirements**: Requires `SYS_ADMIN` capability for huge pages
- **Source Build**: Compiled from official XMRig repository
- **Minimal Base**: Alpine Linux base with only required dependencies
- **No Embedded Secrets**: All pool/wallet info passed as arguments
- **Build Attestation**: GitHub Actions provides build provenance

## Troubleshooting

### Container Exits Immediately
Check logs: `kubectl logs -n mining <pod-name>`
Common issues:
- Missing required arguments (--url, --user)
- Invalid pool URL
- Network connectivity to pool

### Low Hashrate
- Verify huge pages are enabled on host
- Check CPU allocation (increase limits if needed)
- Monitor CPU throttling
- Ensure proper cooling on ARM64 devices

### Exec Format Error
- Verify node architecture is ARM64: `kubectl get nodes -o wide`
- Check image platform: `docker manifest inspect sjgorey/xmrig-arm64:latest`

## Files

- `Dockerfile.xmrig` - Multi-stage build for ARM64 XMRig
- `.github/workflows/build-and-push.yml` - GitHub Actions workflow
- `.github/ACTIONS.md` - GitHub Actions setup guide
- `README.md` - This file

## Hardware Tested

This image has been tested on:
- ODROID-N2+ (ARM Cortex-A55)
- Running k3s Kubernetes

## License

XMRig is licensed under GPL-3.0. This repository contains only the Dockerfile and build configuration.

## References

- [XMRig Official Repository](https://github.com/xmrig/xmrig)
- [XMRig Documentation](https://xmrig.com/docs)
- [Monero P2Pool](https://p2pool.io)

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test the build locally
5. Submit a pull request

## Support

For issues related to:
- **This Docker image**: Open an issue in this repository
- **XMRig itself**: See [XMRig repository](https://github.com/xmrig/xmrig)
- **Kubernetes deployment**: See the home-argocd-app-config repository
