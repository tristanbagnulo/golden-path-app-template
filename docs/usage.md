# Usage Guide

## Developer Steps

1. Navigate to be adjacent to the `infrastructure-platform-devops` repository
2. `git clone https://github.com/tristanbagnulo/golden-path-app-template.git`
3. `mv golden-path-app-template your-application-name`
4. `cd your-application-name`
5. Copy all application files into this directory
6. Add one or more request YAML files into the `infra/requests/` depending on target environment/s (`dev.yaml`, `stage.yaml` or `prod.yaml`)
7. View the examples in `infra/requests/` to understand the available resource types and their parameters
8. Fill in the request YAML file/s accordingly

## For AWS Cloud Deployment

Deploy the application to the AWS EKS service hosted on AWS Cloud services

1. Complete the steps above in [Developer Steps](#developer-steps)
2. Push changes
```bash
git add .
git commit -m "Add application infrastructure"
git push origin main
```
3. The Jenkins pipelines from the `golden-path-app-template` will complete these steps automatically:
    1. Validates the infrastructure request against schema
    2. Calls the infrastructure-platform-devops repository
    3. Runs: `python3 scripts/render.py [Your request] [AWS account details]`
    4. Executes: `terraform plan && terraform apply`
    5. Builds and pushes your app's Docker image
    6. Deploys to Kubernetes with generated infrastructure outputs
    7. Runs integration tests
    8. Notifies of success/failure

## For Local Deployment

Deploy the application to a locally-hosted K8s cluster

### Prerequisites

1. **Local K8s cluster** deployed using `kind` in Docker Desktop. See the `infrastructure-platform-devops` repository for setup instructions.
2. **Python dependencies** for infrastructure generation:
   ```bash
   python3 -m pip install --user --break-system-packages jsonschema pyyaml
   ```
3. **Validate prerequisites:**
```bash
# 1. Ensure kind cluster is running
kind get clusters | grep infra-platform

# 2. Verify tools are working
docker --version
kubectl get nodes
helm version
```

### Steps

**Platform Steps** (handled automatically in Cloud deployment)

1. Navigate to `infrastructure-platform-devops` repository
2. Run the render script to generate the Terraform modules:

```bash
cd infrastructure-platform-devops/runner
python3 scripts/render.py \
    ../../your-application-name/infra/requests/dev.yaml \
    123456789012 us-east-1 \
    arn:aws:iam::123456789012:oidc-provider/example \
    example \
    your-application-name.tf.json
```

**Application Deployment Steps**

3. Navigate to your application repository
4. Build and deploy your application on the local K8s cluster:

```bash
# Build and load Docker image
docker build -t your-app-name .
# If you already have something forwarding port on kubectl
pkill -f "kubectl port-forward"

kind load docker-image your-app-name --name infra-platform

# Deploy with Helm
helm upgrade --install your-app-name charts \
    --set image.repository=your-app-name \
    --set image.tag=latest \
    --set externalSecret.enabled=false \
    --set serviceMonitor.enabled=false \
    --set securityContext.readOnlyRootFilesystem=false \
    --set probes.readinessProbe.path="/health/ready" \
    --set probes.livenessProbe.path="/health/live" \
    --namespace your-app-namespace \
    --create-namespace
```

5. Verify the running application:
```bash
# Check pod status
kubectl get pods -n your-app-namespace

# Check logs
kubectl logs -n your-app-namespace deployment/your-app-name --tail=20

# Port forward to access the app
kubectl port-forward -n your-app-namespace service/your-app-name 8080:80 &
```

6. Visit the app at http://localhost:8080/

## Example: Photo Service Application

For a complete working example, see the photo service demo:

```bash
# Infrastructure request (infra/requests/dev.yaml):
app: photo-service
env: dev
namespace: photo-service
resources:
  - type: s3_bucket
    name: photos
    purpose: uploads
  - type: rds_database
    name: main
    engine: postgres
    size: small
  - type: secret
    name: api-keys
  - type: irsa_role
    name: photo-service
    s3_buckets: [photos]
    rds_databases: [main]
    secrets: [api-keys]
```

This generates AWS infrastructure and Kubernetes deployment automatically!

## Environment-Specific Configuration

The Golden Path template uses environment-specific values to customize deployments:

### File Structure
```
your-app/
├── charts/values.yaml          # Base configuration (defaults)
├── env/dev/values.yaml         # Development overrides
├── env/stage/values.yaml       # Staging overrides
└── env/prod/values.yaml        # Production overrides
```

### How It Works
```bash
# Development deployment uses both files:
helm upgrade --install your-app charts \
  --values charts/values.yaml \      # Base values
  --values env/dev/values.yaml \     # Dev-specific overrides
  --namespace your-app-dev

# Production deployment uses different overrides:
helm upgrade --install your-app charts \
  --values charts/values.yaml \      # Same base values
  --values env/prod/values.yaml \    # Prod-specific overrides
  --namespace your-app-prod
```

### Typical Environment Differences
- **Dev**: 1 replica, debug logging, no auto-scaling
- **Stage**: 2 replicas, info logging, limited auto-scaling  
- **Prod**: 3+ replicas, structured logging, full auto-scaling

The platform handles this automatically based on which environment you're deploying to!
