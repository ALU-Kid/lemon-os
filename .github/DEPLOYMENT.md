# GitHub Actions Deployment Setup

This repository uses GitHub Actions to automatically build and deploy to the VPS on every push to the `main` branch.

## Architecture

1. **Build**: Docker image is built in GitHub Actions and pushed to GitHub Container Registry (GHCR)
2. **Deploy**: VPS pulls the new image from GHCR and recreates containers
3. **Verify**: Health check confirms deployment success

## Required GitHub Secrets

### Go to: Repository Settings → Secrets and variables → Actions → New repository secret

### 1. PAT (Personal Access Token)

- **Description**: GitHub Personal Access Token with `packages:write` and `repo` scopes
- **How to create**:
  1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
  2. Click "Generate new token (classic)"
  3. Give it a name like "CitrineOS Deployment"
  4. Select scopes: `repo` and `write:packages`
  5. Click "Generate token"
  6. Copy the token (you won't see it again!)

### 2. SSH_PRIVATE_KEY

- **Description**: SSH Private Key for VPS access (without BEGIN/END lines)
- **How to get**:

  ```bash
  # Display your private key
  cat ~/.ssh/id_ed25519_hostinger

  # Copy ONLY the content between (but not including) these lines:
  # -----BEGIN OPENSSH PRIVATE KEY-----
  # ... copy this part only ...
  # -----END OPENSSH PRIVATE KEY-----
  ```

- **Format**: Just the base64 content, no header/footer lines

### 3. SSH_HOST

- **Value**: `72.61.143.94`
- **Description**: IP address of your VPS

### 4. SSH_USER

- **Value**: `root`
- **Description**: SSH username for the VPS

### 5. WORK_DIR

- **Value**: `/root/lemon-os/lemon-os/Server`
- **Description**: Working directory on VPS where docker-compose.yml is located

## Quick Setup

### Step 1: Add GitHub Secrets

```bash
# Get your SSH private key (for SSH_PRIVATE_KEY secret)
cat ~/.ssh/id_ed25519_hostinger

# Copy the content WITHOUT the BEGIN/END lines
# Go to GitHub → Your Repo → Settings → Secrets → New repository secret
# Add all 5 secrets listed above
```

### Step 2: Make Repository Public or Configure Package Visibility

For GHCR to work, either:

- Make your repository public, OR
- Configure package visibility:
  1. Go to your GitHub profile → Packages
  2. Find `citrineos` package
  3. Package settings → Change visibility → Public

### Step 3: Push Workflow to GitHub

```bash
cd /home/ira-rutazinda/Documents/lemon-os

# Add workflow files
git add .github/

# Commit
git commit -m "Add CI/CD workflow with GHCR"

# Push to trigger deployment
git push origin main
```

### Step 4: Monitor Deployment

1. Go to GitHub → Actions tab
2. Watch the workflow run
3. Check each step's logs
4. Deployment takes ~5-8 minutes

## How It Works

### Workflow Steps

1. **Check Secrets** - Validates all required secrets are configured
2. **Build Image** - Builds Docker image and pushes to GHCR
3. **Deploy to VPS** - Pulls image on VPS and recreates containers
4. **Verify** - Tests health endpoint

### On First Run

The workflow will automatically update your `docker-compose.yml` on the VPS to use the GHCR image instead of building locally. The original file is backed up as `docker-compose.yml.backup`.

## Deployment Process

### Automatic Deployment

- Every push to `main` triggers deployment
- GitHub Actions builds image and deploys to VPS
- Total time: ~5-8 minutes

### Manual Deployment

1. Go to Actions tab
2. Select "Build and Deploy to VPS"
3. Click "Run workflow"
4. Select branch and click "Run workflow"

## Monitoring

### GitHub Actions

- **Actions tab** → Latest workflow run → View logs

### VPS Logs

```bash
ssh root@72.61.143.94
cd /root/lemon-os/lemon-os/Server

# Check container status
docker compose ps

# View logs
docker compose logs -f citrine

# Check recent deployments
docker images | grep citrineos
```

### Test Endpoints

After deployment:

- **Health**: http://72.61.143.94:8083/health
- **API Docs**: http://72.61.143.94:8083/docs
- **Domain**: https://ocpp.gokabisa.com (once DNS configured)

## Troubleshooting

### Deployment Failed at Build Stage

**Error**: `failed to push to registry`

- **Fix**: Check PAT has `write:packages` scope
- **Fix**: Verify GHCR login successful in build logs

### Deployment Failed at Deploy Stage

**Error**: `Permission denied (publickey)`

- **Fix**: Verify SSH_PRIVATE_KEY is correct (without BEGIN/END lines)
- **Fix**: Ensure public key is in VPS `/root/.ssh/authorized_keys`

**Error**: `cannot pull image`

- **Fix**: Make package public or ensure VPS is logged into GHCR
- **Fix**: Check image name matches: `ghcr.io/[username]/citrineos:latest`

### Health Check Failed

1. Wait 60 seconds and check again: `curl http://72.61.143.94:8083/health`
2. Check logs: `ssh root@72.61.143.94 "cd /root/lemon-os/lemon-os/Server && docker compose logs citrine"`
3. Verify dependencies: `docker compose ps` (RabbitMQ and PostgreSQL should be healthy)

### Rollback to Previous Version

```bash
ssh root@72.61.143.94
cd /root/lemon-os/lemon-os/Server

# List available images
docker images | grep citrineos

# Update to specific version
docker tag ghcr.io/[username]/citrineos:latest ghcr.io/[username]/citrineos:backup
docker compose up -d --force-recreate
```

## Security Best Practices

✅ **DO**:

- Use PAT with minimal required scopes
- Rotate PAT regularly
- Use SSH keys instead of passwords
- Keep secrets in GitHub Secrets (encrypted)
- Enable branch protection on `main`

❌ **DON'T**:

- Commit secrets to repository
- Share PAT or SSH keys
- Use root password authentication
- Disable secret checking in workflow

## Performance

- **Build time**: ~3-5 minutes (on GitHub Actions)
- **Push to GHCR**: ~30 seconds
- **Pull on VPS**: ~10-20 seconds
- **Container restart**: ~30 seconds
- **Health check**: ~10 seconds
- **Total**: ~5-8 minutes

## Next Steps

After deployment:

1. Configure DNS in Cloudflare: `A record: ocpp → 72.61.143.94`
2. Test domain access: https://ocpp.gokabisa.com
3. Set up monitoring and alerts
4. Configure backups
5. Review and update firewall rules
