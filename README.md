# Reusable CI/CD Workflows for Docker + Nomad

Reusable GitHub Actions workflows for building Docker images and deploying to Nomad clusters. Use these workflows in any repository without copying files.

## Quick Start

Create `.github/workflows/deploy.yml` in your application repository:

```yaml
name: Build and Deploy

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    uses: aisystant/ci/.github/workflows/docker-publish.yml@main
    with:
      dockerfile_path: '.'
      platforms: 'linux/amd64'
      push_to_registry: ${{ github.event_name != 'pull_request' }}

  deploy:
    needs: build
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    uses: aisystant/ci/.github/workflows/nomad.yml@main
    with:
      image_tag: 'sha-${{ github.sha }}'
      environment: 'production'
      nomad_job_file: 'nomad-job.hcl'
      nomad_host: ${{ vars.NOMAD_HOST }}
      nomad_user: ${{ vars.NOMAD_USER }}
    secrets:
      NOMAD_SSH_PRIVATE_KEY: ${{ secrets.NOMAD_SSH_PRIVATE_KEY }}
```

## Setup Requirements

### 1. Repository Configuration
Add in your application repository settings → Secrets and variables → Actions:

**Secrets:**
```
NOMAD_SSH_PRIVATE_KEY    # SSH private key for Nomad cluster access
```

**Variables:**
```
NOMAD_HOST              # Nomad cluster hostname/IP  
NOMAD_USER              # SSH username for Nomad cluster
```

### 2. Nomad Job File
Create `nomad-job.hcl` in your repository root:

```hcl
job "my-app" {
  datacenters = ["dc1"]
  type = "service"

  group "web" {
    task "app" {
      driver = "docker"
      
      config {
        # This image line will be automatically updated
        image = "ghcr.io/aisystant/your-repo:latest"
        ports = ["http"]
      }
      
      resources {
        cpu    = 100
        memory = 128
      }
      
      service {
        name = "my-app"
        port = "http"
        
        check {
          type     = "http"
          path     = "/"
          interval = "10s"
          timeout  = "3s"
        }
      }
    }
    
    network {
      port "http" {
        static = 8080
      }
    }
  }
}
```

### 3. Dockerfile
Ensure you have a `Dockerfile` in your repository root.

## Workflow Options

### Docker Build Workflow

**Inputs:**
- `dockerfile_path` (optional): Path to Dockerfile, default: '.'
- `platforms` (optional): Target platforms, default: 'linux/amd64'  
- `push_to_registry` (optional): Push to registry, default: true

**Outputs:**
- `image`: Built image name
- `tags`: Generated tags
- `digest`: Image digest

### Nomad Deploy Workflow  

**Inputs:**
- `image_tag` (optional): Docker image tag, default: 'sha-${{ github.sha }}'
- `environment` (optional): Target environment, default: 'production'
- `nomad_job_file` (optional): Path to Nomad job file, default: 'nomad-job.hcl'

**Required Secrets:**
- `NOMAD_SSH_PRIVATE_KEY`: SSH private key

**Required Variables:**
- `NOMAD_HOST`: Nomad cluster host
- `NOMAD_USER`: SSH username

## Usage Examples

### Basic Usage
```yaml
jobs:
  deploy:
    uses: aisystant/ci/.github/workflows/nomad.yml@main
    secrets:
      NOMAD_SSH_PRIVATE_KEY: ${{ secrets.NOMAD_SSH_PRIVATE_KEY }}
    vars:
      NOMAD_HOST: ${{ vars.NOMAD_HOST }}
      NOMAD_USER: ${{ vars.NOMAD_USER }}
```

### Custom Configuration
```yaml
jobs:
  build:
    uses: aisystant/ci/.github/workflows/docker-publish.yml@main
    with:
      dockerfile_path: './docker/Dockerfile'
      platforms: 'linux/amd64,linux/arm64'
      
  deploy-staging:
    needs: build
    uses: aisystant/ci/.github/workflows/nomad.yml@main
    with:
      image_tag: 'sha-${{ github.sha }}'
      environment: 'staging'
      nomad_job_file: 'deployment/staging.hcl'
    secrets:
      NOMAD_SSH_PRIVATE_KEY: ${{ secrets.NOMAD_SSH_PRIVATE_KEY }}
    vars:
      NOMAD_HOST: ${{ vars.NOMAD_HOST }}
      NOMAD_USER: ${{ vars.NOMAD_USER }}
```

### Manual Deploy Workflow
Create `.github/workflows/manual-deploy.yml`:

```yaml
name: Manual Deploy

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to deploy'
        required: true
        default: 'main'
      environment:
        description: 'Environment'
        required: true
        default: 'production'
        type: choice
        options:
        - production
        - staging

jobs:
  deploy:
    uses: aisystant/ci/.github/workflows/nomad.yml@main
    with:
      image_tag: ${{ inputs.image_tag }}
      environment: ${{ inputs.environment }}
    secrets:
      NOMAD_SSH_PRIVATE_KEY: ${{ secrets.NOMAD_SSH_PRIVATE_KEY }}
    vars:
      NOMAD_HOST: ${{ vars.NOMAD_HOST }}
      NOMAD_USER: ${{ vars.NOMAD_USER }}
```

## How It Works

1. **Build Phase**: Docker workflow builds and pushes image to GHCR
2. **Deploy Phase**: Nomad workflow updates job file with new image tag
3. **Validation**: Validates Nomad job and checks image accessibility  
4. **Deployment**: Executes deployment with health monitoring
5. **Verification**: Confirms successful deployment

## Benefits of Reusable Workflows

✅ **No file copying** - Reference workflows directly  
✅ **Always up-to-date** - Use latest improvements automatically  
✅ **Version control** - Pin to specific versions with `@v1.0`  
✅ **Centralized** - Update workflows in one place  
✅ **Flexible** - Customize with inputs and secrets

## Troubleshooting

### Build Issues
- Check Dockerfile exists and is valid
- Verify all dependencies are available
- Review build logs in GitHub Actions

### Deploy Issues  
- Verify secrets are configured correctly
- Check Nomad cluster connectivity
- Ensure job file syntax is valid
- Review deployment logs for specific errors

### Common Fixes
```bash
# Test Nomad job locally
nomad job validate nomad-job.hcl

# Check SSH connectivity  
ssh -i key.pem user@nomad-host

# Verify image exists
docker pull ghcr.io/aisystant/your-repo:tag
```

## Workflow Versioning

Pin workflows to specific versions for stability:

```yaml
# Use latest (recommended for development)
uses: aisystant/ci/.github/workflows/nomad.yml@main

# Use specific version (recommended for production)  
uses: aisystant/ci/.github/workflows/nomad.yml@v1.0

# Use specific commit
uses: aisystant/ci/.github/workflows/nomad.yml@abc123
```