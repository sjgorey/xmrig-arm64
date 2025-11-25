# GitHub Actions Setup for XMRig ARM64

This document explains how to set up GitHub Actions for automatic Docker image building and publishing of the ARM64 XMRig miner.

## Required Secrets

Before the workflow can run successfully, you need to add the following secrets to your GitHub repository:

### Setting up Docker Hub Secrets

1. Go to your repository on GitHub: `https://github.com/sjgorey/xmrig-arm64`
2. Click on **Settings** tab
3. In the left sidebar, click **Secrets and variables** → **Actions**
4. Click **New repository secret** and add:

#### DOCKER_USERNAME
- **Name**: `DOCKER_USERNAME`
- **Value**: Your Docker Hub username (e.g., `sjgorey`)

#### DOCKER_PASSWORD
- **Name**: `DOCKER_PASSWORD`  
- **Value**: Your Docker Hub access token (recommended) or password

### Creating a Docker Hub Access Token (Recommended)

Instead of using your password, create an access token:

1. Log in to [Docker Hub](https://hub.docker.com)
2. Go to **Account Settings** → **Security**
3. Click **New Access Token**
4. Give it a name: `github-actions-xmrig-arm64`
5. Select permissions: **Read & Write**
6. Copy the generated token
7. Use this token as the `DOCKER_PASSWORD` secret value

## Workflow Features

The GitHub Actions workflow (`build-and-push.yml`) includes:

### Triggers
- **Push to main**: Builds and pushes new image with `latest` tag
- **Push tags**: Builds versioned releases (e.g., `v6.21.0`)
- **Pull requests**: Runs tests only (no push)

### Testing Stage
- Validates Dockerfile exists
- Checks Dockerfile syntax with hadolint
- Ensures build prerequisites are met

### Build and Push Stage
- ARM64 platform build using QEMU emulation
- Automatic tagging based on Git refs
- Docker layer caching for faster builds
- Build attestation for security

### Generated Tags

The workflow automatically creates these Docker image tags:

| Trigger | Tags Generated |
|---------|----------------|
| Push to main | `latest`, `main-<sha>` |
| Tag `v6.21.0` | `v6.21.0`, `6.21`, `6`, `latest` |
| PR #123 | `pr-123` (test build only, not pushed) |

## XMRig Version

The current Dockerfile builds XMRig version **v6.21.0**. To update to a newer version:

1. Edit `Dockerfile.xmrig`
2. Change the branch version: `--branch v6.21.0` to your desired version
3. Commit and push
4. Tag the commit with the version number:
   ```bash
   git tag v6.21.0
   git push origin v6.21.0
   ```

## Manual Workflow Trigger

You can also manually trigger the workflow:

1. Go to **Actions** tab in your GitHub repository
2. Select **Build and Push XMRig Docker Image** workflow
3. Click **Run workflow**
4. Choose the branch and click **Run workflow**

## Monitoring Builds

### View Build Status
- Go to the **Actions** tab in your repository
- Click on any workflow run to see detailed logs
- Check the **Jobs** section for test and build results

### Build Badges

Add a build status badge to your README:

```markdown
![Build Status](https://github.com/sjgorey/xmrig-arm64/actions/workflows/build-and-push.yml/badge.svg)
```

## Troubleshooting

### Common Issues

1. **Authentication Failed**
   - Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD` secrets are correct
   - If using 2FA, ensure you're using an access token, not your password

2. **Build Fails on ARM64**
   - The workflow uses QEMU for ARM64 emulation
   - ARM64 builds take longer than AMD64 builds (15-30 minutes)
   - Check that all dependencies support ARM64

3. **XMRig Compilation Errors**
   - Ensure the XMRig version tag exists in the repository
   - Check that all Alpine packages are available
   - Review build logs for specific compilation errors

### Build Time Expectations

- **First build**: 20-30 minutes (compiling XMRig from source)
- **Subsequent builds**: 5-10 minutes (with layer caching)

### Debugging Steps

1. **Check workflow file syntax**:
   ```bash
   # Install GitHub CLI (if not already installed)
   gh workflow view build-and-push.yml
   ```

2. **Test Docker build locally** (requires ARM64 or QEMU):
   ```bash
   docker buildx build --platform linux/arm64 -f Dockerfile.xmrig -t test-xmrig .
   ```

3. **Verify XMRig version exists**:
   ```bash
   git ls-remote --tags https://github.com/xmrig/xmrig.git | grep v6.21.0
   ```

## Security Considerations

- **Use access tokens** instead of passwords for Docker Hub
- **Limit token permissions** to only what's needed (Read & Write)
- **Rotate tokens periodically** for security
- **Monitor workflow runs** for any suspicious activity
- **Review XMRig updates** before building new versions

## Performance Optimization

### Huge Pages Support
The built XMRig binary includes huge pages support for improved mining performance:
- Compiled with `HWLOC` support
- Requires host system to have huge pages enabled
- Mount `/dev/hugepages` in your container

### Build Optimizations
The Dockerfile includes:
- Multi-stage build to minimize image size
- Stripped binary for smaller footprint
- Alpine Linux base for minimal dependencies

## Customization

### Change XMRig Build Options

Edit `Dockerfile.xmrig` to modify CMake options:

```dockerfile
cmake .. -DWITH_CN_GPU=OFF -DWITH_OPENCL=OFF -DWITH_CUDA=OFF
```

Available options:
- `WITH_HWLOC=ON/OFF` - Hardware locality support (default: enabled in base image)
- `WITH_CN_GPU=ON/OFF` - CryptoNight GPU support
- `WITH_OPENCL=ON/OFF` - OpenCL support
- `WITH_CUDA=ON/OFF` - CUDA support

### Add Additional Platforms

Currently builds for ARM64 only. To add AMD64:

```yaml
platforms: linux/amd64,linux/arm64
```

Note: AMD64 builds will be faster and may not require QEMU.

## Deployment

After the image is built and pushed, update your Kubernetes DaemonSet:

```yaml
image: sjgorey/xmrig-arm64:latest
# or use a specific version
image: sjgorey/xmrig-arm64:v6.21.0
```

Then apply the configuration:
```bash
kubectl apply -f infrastructure/mining/03-daemonset.yaml
```
