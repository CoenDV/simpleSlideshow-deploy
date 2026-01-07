# simpleSlideshow-deploy

Kubernetes deployment configuration using Kustomize for the Group1 microservices platform.

## Structure

Each service follows this layout:
```
ServiceName/
├── base/                    # Base Kubernetes manifests
│   ├── deployment-*.yaml    # Deployment configuration
│   ├── service-*.yaml       # Service definition
│   ├── route-*.yaml         # OpenShift Route
│   └── kustomization.yaml   # Base kustomization
└── overlays/                # Environment-specific configs
    ├── prod/                # Production environment
    └── test/                # Test environment
```

## Services

- **ChatService** - Real-time chat and messaging
- **CommunityService** - Community interactions (likes, favorites, comments, shares, watch history)
- **NotificationService** - Push notifications and alerts
- **RecommendationService** - Personalized content recommendations using collaborative filtering
- **UserService** - User authentication and profile management
- **VideoService** - Video upload, streaming, and downloads with Mux integration (includes PostgreSQL)

## Environments

| Environment | Namespace | Image Tag | Branch |
|------------|-----------|-----------|--------|
| **test** | `projectgroup1-test` | `:dev` | dev |
| **prod** | `projectgroup1-prod` | `:main` | main |

## Deployment

### Using Kustomize
```bash
# Deploy to test
kubectl apply -k ServiceName/overlays/test

# Deploy to prod
kubectl apply -k ServiceName/overlays/prod
```

### Using ArgoCD
Point ArgoCD applications to:
- Test: `ServiceName/overlays/test`
- Prod: `ServiceName/overlays/prod`

## Service Details

### VideoService

VideoService provides video management with Mux integration for streaming and downloads.

**Features:**
- Dual-asset strategy: Clean videos for streaming + watermarked videos for downloads
- Private video streaming with JWT-signed URLs
- Mux webhook integration for asset processing
- Automatic sync to RecommendationService when videos are ready
- PostgreSQL database with persistent storage

**Configuration:**

The deployment includes:
- PostgreSQL database with PVC (Persistent Volume Claim)
- Ngrok service for webhook testing in development
- Health checks (liveness and readiness probes)
- Horizontal Pod Autoscaler (HPA) for scaling

**Environment Variables (configured in deployment):**
- `WATERMARK_ENABLED` - Enable/disable watermarks (default: true)
- `WATERMARK_IMAGE_URL` - URL to watermark image (must be publicly accessible HTTPS)
- `WATERMARK_POSITION` - Position: top-left, top-right, bottom-left, bottom-right
- `WATERMARK_WIDTH` - Watermark width in pixels
- `WATERMARK_HEIGHT` - Watermark height in pixels
- `WATERMARK_OPACITY` - Opacity from 0.0 to 1.0

**Secrets Required:**
- `MUX_TOKEN_ID` - Mux API token ID
- `MUX_TOKEN_SECRET` - Mux API token secret
- `MUX_WEBHOOK_SECRET` - Mux webhook signature verification
- `MUX_SIGNING_KEY_ID` - For generating JWT-signed playback URLs
- `MUX_SIGNING_KEY_PRIVATE_KEY` - Base64-encoded private key
- `SERVICE_TOKEN` - Shared token for RecommendationService sync
- `RECOMMENDATION_SERVICE_URL` - RecommendationService endpoint
- `KEYCLOAK_URL`, `KEYCLOAK_REALM`, `KEYCLOAK_CLIENT_ID`, `KEYCLOAK_AUDIENCE` - Authentication

### RecommendationService

Provides personalized video recommendations using collaborative filtering and content-based algorithms.

**Features:**
- LightFM-based collaborative filtering with weighted interactions
- Multiple recommendation sources: collaborative, trending, popular, new content
- Category and tag-based filtering
- Automatic model training jobs
- Video sync endpoints for real-time updates

**Configuration:**

The deployment includes:
- PostgreSQL database with PVC
- Prometheus metrics endpoint for monitoring
- Health checks and readiness probes

**Secrets Required:**
- `SERVICE_TOKEN` - Shared token for authenticating sync requests from other services
- Database credentials (DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD)

### Inter-Service Communication

**VideoService → RecommendationService:**
- Automatically syncs new videos when assets are ready
- Sends video ID, categories, and tags
- Authenticated via SERVICE_TOKEN

**CommunityService → RecommendationService:**
- Syncs user interactions (likes, favorites, shares, watches, comments)
- Real-time updates for recommendation algorithm
- Authenticated via SERVICE_TOKEN

**VideoService → UserService:**
- Fetches user preferences (interests, discovery radius)
- Used for personalizing video feed
- Authenticated via JWT token forwarding

## Secrets Management

Secrets are stored in Kubernetes Secret objects for each service. Example structure:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: videoservice-secrets
type: Opaque
stringData:
  MUX_TOKEN_ID: "your_token_id"
  MUX_TOKEN_SECRET: "your_token_secret"
  MUX_WEBHOOK_SECRET: "your_webhook_secret"
  MUX_SIGNING_KEY_ID: "your_signing_key_id"
  MUX_SIGNING_KEY_PRIVATE_KEY: "base64_encoded_key"
  SERVICE_TOKEN: "shared_service_token"
  # ... additional secrets
```

**Important:** The `SERVICE_TOKEN` must be identical across:
- VideoService
- RecommendationService  
- CommunityService

This ensures proper authentication for inter-service sync operations.

## Updating Deployments

### Updating Environment Variables

Edit the deployment YAML in `base/ServiceName/deployment-servicename.yaml`:

```yaml
env:
  - name: WATERMARK_ENABLED
    value: "true"
  - name: WATERMARK_IMAGE_URL
    value: "https://cdn.example.com/watermark.png"
```

Then apply the changes:
```bash
kubectl apply -k base/ServiceName/
```

### Updating Secrets

Secrets are managed separately and referenced in deployments. Update via:

```bash
# Create or update secret
kubectl create secret generic videoservice-secrets \
  --from-literal=MUX_TOKEN_ID=your_token_id \
  --from-literal=SERVICE_TOKEN=shared_token \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Rolling Deployments

Changes to deployment configurations trigger rolling updates automatically:
- New pods are created with updated config
- Old pods are terminated after new ones are ready
- Zero downtime for most configuration changes

## Monitoring

### Health Endpoints

All services expose health check endpoints:
- `/health/live` or `/healthz` - Liveness probe
- `/health/ready` or `/ready` - Readiness probe with dependency checks

### Metrics

Services expose Prometheus metrics:
- **VideoService**: `/metrics` on port 9090
- **RecommendationService**: `/metrics` on port 8084

ServiceMonitor resources are configured for Prometheus autodiscovery.

## Troubleshooting

### Video Sync Issues

If videos aren't appearing in recommendations:

1. **Check VideoService logs for sync errors:**
   ```bash
   oc logs deployment/videoservice -n projectgroup1-prod | grep -i "sync\|recommendation"
   ```

2. **Verify SERVICE_TOKEN matches across services:**
   ```bash
   oc get secret videoservice-secrets -o jsonpath='{.data.SERVICE_TOKEN}' | base64 -d
   oc get secret recommendationservice-secrets -o jsonpath='{.data.SERVICE_TOKEN}' | base64 -d
   ```

3. **Check RecommendationService sync endpoint:**
   ```bash
   oc logs deployment/recommendationservice -n projectgroup1-prod | grep -i "sync"
   ```

### Watermark Not Appearing

If downloads have no watermark:

1. **Verify WATERMARK_IMAGE_URL is accessible:**
   ```bash
   curl -I https://your-watermark-url.png
   ```

2. **Check VideoService logs during upload:**
   ```bash
   oc logs deployment/videoservice -n projectgroup1-prod | grep -i "watermark"
   ```

3. **Ensure environment variables are set:**
   ```bash
   oc get deployment videoservice -o yaml | grep -A5 WATERMARK
   ```

### Database Connection Issues

If services can't connect to PostgreSQL:

1. **Check database pod status:**
   ```bash
   oc get pods | grep postgresql
   ```

2. **Verify database credentials:**
   ```bash
   oc get secret videoservice-secrets -o yaml
   ```

3. **Test database connectivity:**
   ```bash
   oc exec -it deployment/videoservice-postgresql -- psql -U postgres -d videoservice -c "SELECT 1"
   ```
