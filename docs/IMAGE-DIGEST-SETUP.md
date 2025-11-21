# Automated Image Digest Updates

This document explains how to set up automated image digest updates for each service.

## Overview

When a new Docker image is pushed to the `main` or `dev` branch, a GitHub Actions workflow automatically:
1. Pulls the new image
2. Extracts its digest (SHA256 hash)
3. Updates the Kustomize overlay in this repo
4. Commits and pushes the change
5. ArgoCD detects the manifest change and deploys the new image

## Setup Instructions

### 1. Create GitHub Personal Access Token (PAT)

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Name it: `DEPLOY_REPO_TOKEN`
4. Select scopes:
   - `repo` (Full control of private repositories)
5. Generate and copy the token

### 2. Add Secret to Each Service Repository

For each service repository (VideoService, ChatService, UserService, etc.):

1. Go to repository Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `DEPLOY_REPO_TOKEN`
4. Value: Paste the PAT you created
5. Click "Add secret"

### 3. Add Workflow to Each Service Repository

Copy the template from `docs/service-workflow-template.yml` and customize it:

**For VideoService:**
```yaml
service-name: VideoService
image-name: ghcr.io/coendv/group1-videoservice
```

**For ChatService:**
```yaml
service-name: ChatService
image-name: ghcr.io/coendv/group1-chatservice
```

**For UserService:**
```yaml
service-name: UserService
image-name: ghcr.io/coendv/group1-userservice
```

And so on for each service.

Save as `.github/workflows/update-deployment.yml` in each service repo.

### 4. Update Workflow Trigger

In the workflow file, update the `workflows` field to match your actual Docker build workflow name:

```yaml
workflows: ["CI/CD"]  # Replace with your actual workflow name
```

## How It Works

### Image Tags vs Digests

**Before (using tags):**
```yaml
images:
  - name: ghcr.io/.../group1-videoservice
    newTag: main  # Tag doesn't change, K8s won't update
```

**After (using digests):**
```yaml
images:
  - name: ghcr.io/.../group1-videoservice
    digest: sha256:abc123...  # Digest changes with each build
```

### Workflow Sequence

```
Service Repo (main branch)
  ↓
Docker Build & Push
  ↓
Update Deployment Workflow
  ↓
Pull new image & get digest
  ↓
Update simpleSlideshow-deploy repo
  ↓
Commit & Push changes
  ↓
ArgoCD detects change
  ↓
Deploy new pods
```

## Testing

1. Push a change to a service's `main` branch
2. Wait for Docker build to complete
3. Check the Actions tab in the service repo
4. Verify "Update Deployment Manifest" workflow runs
5. Check this repo for a new commit updating the digest
6. Verify ArgoCD deploys the new image

## Troubleshooting

**Workflow doesn't trigger:**
- Check the workflow name in `workflows:` matches your build workflow
- Verify the build workflow completed successfully
- Check the branch filter matches (`main` or `dev`)

**Permission denied:**
- Verify `DEPLOY_REPO_TOKEN` secret exists in service repo
- Check PAT has `repo` scope
- Ensure PAT hasn't expired

**No changes committed:**
- Check if digest actually changed (rebuild vs cache)
- Verify workflow has write access to deployment repo

## Alternative: Manual Digest Update

To manually update a digest:

```bash
# Pull the latest image
docker pull ghcr.io/coendv/group1-videoservice:main

# Get the digest
docker inspect ghcr.io/coendv/group1-videoservice:main \
  --format='{{index .RepoDigests 0}}'

# Update kustomization.yaml with the digest
# Edit VideoService/overlays/prod/kustomization.yaml
```
