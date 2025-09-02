# Golden Path App Template

> A comprehensive template for deploying cloud-native applications on EKS using the Golden Path methodology.

This repository provides a production-ready Helm chart and CI/CD pipeline for deploying applications securely and reliably onto EKS clusters. It's designed to work with the `infrastructure-platform-devops` repository to provide a complete developer platform experience.

## 🏗️ Architecture Overview

The Golden Path App Template implements a **platform-as-a-service** approach where:

- **Platform Team** manages the underlying infrastructure (EKS clusters, add-ons, networking, security)
- **Application Teams** use this template to deploy applications with minimal configuration
- **Terraform Service Catalog** provisions AWS resources through standardized modules
- **External Secrets** operator manages secret injection from AWS SSM Parameter Store

> 📋 **Detailed Architecture**: See [docs/architecture.md](docs/architecture.md) for comprehensive architecture documentation and diagrams.

## 📂 Repository Structure

```
golden-path-app-template/
├── charts/myapp/                    # Helm chart for application deployment
│   ├── Chart.yaml                   # Chart metadata
│   ├── values.yaml                  # Default configuration values
│   └── templates/                   # Kubernetes manifest templates
│       ├── _helpers.tpl             # Template helper functions
│       ├── deployment.yaml          # Application deployment
│       ├── service.yaml             # Kubernetes service
│       ├── ingress.yaml             # Ingress configuration
│       ├── hpa.yaml                 # Horizontal Pod Autoscaler
│       ├── pdb.yaml                 # Pod Disruption Budget
│       ├── networkpolicy.yaml       # Network security policies
│       ├── serviceaccount.yaml      # Service account with IRSA
│       ├── externalsecret.yaml      # External secrets configuration
│       ├── servicemonitor.yaml      # Prometheus monitoring
│       ├── job.yaml                 # One-time job template
│       └── cronjob.yaml             # Scheduled job template
├── env/                             # Environment-specific configurations
│   ├── dev/values.yaml              # Development overrides
│   ├── stage/values.yaml            # Staging overrides
│   └── prod/values.yaml             # Production overrides
├── infra/requests/                  # Infrastructure request files
│   ├── dev.yaml                     # Development resources
│   ├── stage.yaml                   # Staging resources
│   └── prod.yaml                    # Production resources
├── Jenkinsfile                      # CI/CD pipeline definition
└── README.md                        # This file
```

## 🚀 Quick Start

### Prerequisites
- EKS cluster with platform add-ons installed:
  - Ingress NGINX Controller
  - cert-manager for TLS certificates
  - External Secrets Operator with ClusterSecretStore named `aws-ssm`
  - kube-prometheus-stack for monitoring
- Access to `infrastructure-platform-devops` repository and runner job
- Jenkins with pipeline configured

### 1. Fork and Customize

1. Fork this repository for your application
2. Update the application name in relevant files:
   - `charts/myapp/Chart.yaml` - Update `name` and `description`
   - `charts/myapp/values.yaml` - Update `image.repository`
   - `infra/requests/*.yaml` - Update `app` name

### 2. Configure Infrastructure Requirements

Edit the infrastructure request files in `infra/requests/` to specify the AWS resources your application needs:

```yaml
# infra/requests/dev.yaml
app: your-app-name
env: dev
namespace: your-app-name

resources:
  # S3 bucket for file uploads
  - type: s3_bucket
    name: uploads
    purpose: uploads
    public_access: false

  # Database (optional)
  - type: rds_database
    name: main
    engine: postgres
    size: small

  # Secrets for API keys
  - type: secret
    name: app-secrets
    description: "Application secrets and API keys"
      
  # IRSA role for AWS permissions
  - type: irsa_role
    name: your-app-name
    s3_buckets: [uploads]
    rds_databases: [main]
    secrets: [app-secrets]
```

### 3. Customize Application Configuration

Update the Helm values files for each environment:

```yaml
# env/dev/values.yaml
ingress:
  hosts:
    - host: your-app.dev.example.com

resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"

env:
  LOG_LEVEL: "debug"
  APP_ENV: "development"
```

### 4. Deploy

#### Manual Deployment
```bash
# Deploy to development with infrastructure
helm upgrade --install myapp charts/myapp \
  --namespace myapp --create-namespace \
  --values charts/myapp/values.yaml \
  --values env/dev/values.yaml
```

#### Pipeline Deployment
Trigger the Jenkins pipeline with your desired parameters:
- **Environment**: `dev`, `stage`, or `prod`
- **Action**: `deploy`, `request-infra`, or `deploy-with-infra`
- **Image Tag**: Your container image tag
- **Dry Run**: Preview changes without applying

## 📋 Configuration Reference

### Application Configuration (`values.yaml`)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of pod replicas | `3` |
| `image.repository` | Container image repository | `ghcr.io/your-org/myapp` |
| `image.tag` | Container image tag | `""` (uses Chart.AppVersion) |
| `resources.requests` | CPU/Memory requests | `100m/128Mi` |
| `resources.limits` | CPU/Memory limits | `500m/512Mi` |
| `hpa.enabled` | Enable horizontal pod autoscaling | `true` |
| `hpa.minReplicas` | Minimum pod replicas | `3` |
| `hpa.maxReplicas` | Maximum pod replicas | `15` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.hosts` | Ingress hostnames | `myapp.example.com` |
| `networkPolicy.enabled` | Enable network policies | `true` |
| `serviceMonitor.enabled` | Enable Prometheus monitoring | `true` |

### Infrastructure Configuration (`infra/requests/*.yaml`)

#### Supported Resource Types

| Type | Description | Configuration Options |
|------|-------------|----------------------|
| `s3_bucket` | S3 bucket with security defaults | `purpose` (uploads/static/backups), `public_access` |
| `rds_database` | RDS database (Postgres/MySQL) | `engine`, `size` (small/medium/large) |
| `secret` | AWS Secrets Manager | `description` |
| `irsa_role` | IAM role for service accounts | `s3_buckets`, `rds_databases`, `secrets` (lists of resource names) |

## 🔐 Security Features

### Built-in Security Controls

- **Pod Security Context**: Non-root user, read-only filesystem, dropped capabilities
- **Network Policies**: Zero-trust networking with explicit allow rules
- **IRSA (IAM Roles for Service Accounts)**: Pod-level AWS permissions without long-lived credentials
- **External Secrets**: Automatic secret injection from AWS SSM Parameter Store
- **Ingress Security**: SSL termination, rate limiting, ModSecurity WAF

### Security Best Practices

1. **Least Privilege**: IRSA roles grant minimal required permissions
2. **Secret Management**: No secrets in Git or Terraform state
3. **Network Segmentation**: NetworkPolicies enforce pod-to-pod communication rules
4. **Container Security**: Distroless images, non-root execution, read-only filesystem
5. **Compliance**: Audit logging, encryption at rest and in transit

## 📊 Monitoring and Observability

### Built-in Monitoring

- **Prometheus Metrics**: ServiceMonitor for automatic scraping
- **Health Checks**: Readiness and liveness probes with configurable paths
- **Distributed Tracing**: Ready for OpenTelemetry integration
- **Structured Logging**: JSON logs with correlation IDs

### Monitoring Configuration

```yaml
serviceMonitor:
  enabled: true
  path: /metrics
  interval: 30s
  scrapeTimeout: 10s

probes:
  readinessProbe:
    path: "/health/ready"
    initialDelaySeconds: 5
    periodSeconds: 5
  livenessProbe:
    path: "/health/live"
    initialDelaySeconds: 10
    periodSeconds: 10
```

## 🔄 CI/CD Pipeline

### Pipeline Stages

1. **Validation**: Helm chart linting and Kubernetes manifest validation
2. **Infrastructure**: Provision AWS resources via Terraform service catalog
3. **Deployment**: Deploy application with Helm
4. **Testing**: Smoke tests and health checks
5. **Approval**: Manual approval gate for production deployments

### Pipeline Parameters

| Parameter | Description | Options |
|-----------|-------------|---------|
| `ENVIRONMENT` | Target environment | `dev`, `stage`, `prod` |
| `ACTION` | Deployment action | `deploy`, `request-infra`, `deploy-with-infra` |
| `IMAGE_TAG` | Container image tag | Any valid tag |
| `DRY_RUN` | Preview changes without applying | `true`, `false` |

## 🎯 Environment-Specific Configuration

### Development (`env/dev/values.yaml`)
- Lower resource limits
- Debug logging enabled
- Relaxed security policies
- Fast iteration cycle

### Staging (`env/stage/values.yaml`)
- Production-like configuration
- Integration testing
- Performance testing
- User acceptance testing

### Production (`env/prod/values.yaml`)
- High availability (multi-AZ)
- Production resource limits
- Strict security policies
- Comprehensive monitoring

## 🚨 Troubleshooting

### Common Issues

#### Deployment Fails
```bash
# Check pod status
kubectl get pods -n myapp

# Check pod logs
kubectl logs -n myapp deployment/myapp

# Check events
kubectl get events -n myapp --sort-by='.lastTimestamp'
```

#### External Secrets Not Working
```bash
# Check ExternalSecret status
kubectl get externalsecret -n myapp

# Check ClusterSecretStore
kubectl get clustersecretstore aws-ssm

# Verify IRSA permissions
kubectl describe serviceaccount myapp -n myapp
```

#### Infrastructure Provisioning Fails
```bash
# Check infrastructure runner job logs
# In infrastructure-platform-devops repository
kubectl logs -n platform-system job/runner-myapp-dev
```

## 🤝 Contributing

### Making Changes

1. Create a feature branch from `main`
2. Update configuration files as needed
3. Test changes in development environment
4. Submit pull request with description of changes
5. Platform team will review and provide feedback

### Platform Improvements

To suggest improvements to the golden path template:

1. Open an issue in this repository
2. Propose changes with use case and benefits
3. Platform team will evaluate and prioritize

---

**Maintained by the Platform Engineering Team**
