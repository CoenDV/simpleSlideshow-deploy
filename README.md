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

- **ChatService** - Chat functionality
- **CommunityService** - Community management
- **NotificationService** - Notifications
- **RecommendationService** - Content recommendations
- **UserService** - User management
- **VideoService** - Video management (includes PostgreSQL)

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

## VideoService Notes

VideoService includes a PostgreSQL database with:
- OpenShift-compatible security context
- Persistent volume for data storage
- Database credentials via Kubernetes Secrets

Environment variables are configured for database connectivity.
