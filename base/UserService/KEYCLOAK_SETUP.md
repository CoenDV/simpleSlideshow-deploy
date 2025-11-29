# KeyCloak Setup for UserService

This document describes the KeyCloak deployment setup for the UserService in OpenShift.

## Overview

KeyCloak is deployed as a separate container that provides authentication and authorization services for the UserService and other microservices in the SimpleSlideshow application.

## Components

### 1. KeyCloak Deployment (`deployment-keycloak.yaml`)

The KeyCloak deployment runs KeyCloak 26.0 from the official Quay.io image with the following configuration:

- **Image**: `quay.io/keycloak/keycloak:26.0`
- **Command**: `start-dev --import-realm` (development mode with realm import capability)
- **Port**: 8080
- **Database**: PostgreSQL (shared with UserService)
- **Health Probes**: 
  - Liveness: Checks root path `/` after 60s
  - Readiness: Checks `/realms/master` after 30s

### 2. KeyCloak Service (`service-keycloak.yaml`)

ClusterIP service exposing KeyCloak on port 8080 for internal service-to-service communication.

### 3. KeyCloak Route (`route-keycloak.yaml`)

OpenShift route providing external HTTPS access to the KeyCloak admin console with edge TLS termination.

## Environment Variables

### KeyCloak Container

- `KC_BOOTSTRAP_ADMIN_USERNAME`: admin (default admin username)
- `KC_BOOTSTRAP_ADMIN_PASSWORD`: admin (default admin password - **CHANGE IN PRODUCTION**)
- `KC_DB`: postgres (database type)
- `KC_DB_URL`: jdbc:postgresql://userservice-postgresql:5431/userservice
- `KC_DB_USERNAME`: postgres
- `KC_DB_PASSWORD`: postgres (**CHANGE IN PRODUCTION**)
- `KC_PROXY`: edge (KeyCloak runs behind a reverse proxy)
- `KC_HOSTNAME_STRICT`: false (allows flexible hostname configuration)
- `KC_HOSTNAME_STRICT_HTTPS`: false (allows HTTP in development)
- `KC_HTTP_ENABLED`: true (enables HTTP listener)

### UserService Container

The UserService requires the following KeyCloak environment variables:

- `KEYCLOAK_URL`: https://userservice-keycloak-projectgroup1-prod.apps.inholland-minor.openshift.eu (internal KeyCloak service URL)
- `KEYCLOAK_REALM`: simpleslideshow (realm name)
- `KEYCLOAK_CLIENT_ID`: simpleslideshow-client (OAuth client ID)
- `KEYCLOAK_CLIENT_SECRET`: changeme (**CHANGE IN PRODUCTION**)
- `KEYCLOAK_ADMIN_USER`: admin (KeyCloak admin username)
- `KEYCLOAK_ADMIN_PASS`: admin (**CHANGE IN PRODUCTION**)

## Initial Setup

### 1. Deploy KeyCloak

The KeyCloak deployment is included in the UserService kustomization and will be deployed automatically:

```bash
oc apply -k overlays/test/UserService
```

### 2. Access KeyCloak Admin Console

After deployment, get the KeyCloak route URL:

```bash
oc get route userservice-keycloak -n projectgroup1-test
```

Access the admin console at the route URL and log in with:
- Username: admin
- Password: admin

### 3. Configure Realm

The KeyCloak instance is configured to import realms on startup. To set up the `simpleslideshow` realm:

1. Create a new realm named `simpleslideshow`
2. Create a client named `simpleslideshow-client`
3. Configure the client:
   - Client Protocol: openid-connect
   - Access Type: confidential
   - Valid Redirect URIs: Add your application URLs
   - Web Origins: Add allowed CORS origins
4. Note the client secret from the Credentials tab
5. Update the `KEYCLOAK_CLIENT_SECRET` environment variable in the UserService deployment

### 4. Create Users

Users can be created through:
- KeyCloak admin console
- UserService API (which creates users in both the UserService database and KeyCloak)

## Database

KeyCloak shares the PostgreSQL database with UserService:
- Host: `userservice-postgresql`
- Port: 5431
- Database: `userservice`
- KeyCloak creates its own tables with `KC_` prefix

## Security Considerations

⚠️ **IMPORTANT**: The default credentials are for development only. For production:

1. Change `KC_BOOTSTRAP_ADMIN_PASSWORD` to a strong password
2. Change `KC_DB_PASSWORD` to a secure database password
3. Update `KEYCLOAK_CLIENT_SECRET` with the actual client secret from KeyCloak
4. Use Kubernetes secrets instead of plaintext environment variables
5. Enable HTTPS strict mode (`KC_HOSTNAME_STRICT_HTTPS: true`)
6. Use production mode (`start` command instead of `start-dev`)
7. Configure proper redirect URIs and CORS origins

## Integration with Other Services

To integrate other services (ChatService, VideoService, etc.) with KeyCloak authentication:

1. Add the following environment variables to the service deployment:
   ```yaml
   - name: KEYCLOAK_URL
     value: http://keycloak:8080
   - name: KEYCLOAK_REALM
     value: simpleslideshow
   - name: KEYCLOAK_CLIENT_ID
     value: simpleslideshow-client
   ```

2. Update the service code to validate JWT tokens from KeyCloak

## Troubleshooting

### KeyCloak Pod Not Starting

Check the logs:
```bash
oc logs deployment/userservice-keycloak -n projectgroup1-test
```

Common issues:
- Database connection issues: Verify PostgreSQL is running and accessible
- Memory issues: KeyCloak requires at least 512Mi memory

### Cannot Access Admin Console

Check the route:
```bash
oc get route userservice-keycloak -n projectgroup1-test
oc describe route userservice-keycloak -n projectgroup1-test
```

### UserService Cannot Connect to KeyCloak

Verify the KeyCloak service is accessible:
```bash
oc get svc userservice-keycloak -n projectgroup1-test
```

Test connectivity from UserService pod:
```bash
oc exec deployment/userservice-userservice -n projectgroup1-test -- curl -v http://keycloak:8080
```

## References

- [KeyCloak Documentation](https://www.keycloak.org/documentation)
- [KeyCloak on OpenShift](https://www.keycloak.org/getting-started/getting-started-openshift)
- [KeyCloak Server Configuration](https://www.keycloak.org/server/all-config)
