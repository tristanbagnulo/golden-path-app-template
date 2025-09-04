# Golden Path Platform Features

## Example: Sarah's Photo Service

### Sarah's Application Requirements

* Blob storage for photos (AWS S3 bucket/s)
* RDB for user and metadata (AWS RDS)
* Secret storage for service API keys to access Blob storage and RDB (AWS Secrets)
* Credentials for AWS service access (AWS IRSA Role)

### Steps for Sarah

1. **Clone the template**: `git clone golden-path-app-template photo-service`
2. **Add her application code**: Copy `app.py`, `Dockerfile`, `requirements.txt`
3. **Customize infrastructure requests**: Edit `infra/requests/dev.yaml` with her needs
4. **Deploy**: `git add . && git commit -m "Deploy photo service" && git push`

### Automatic fulfillment code provides the following

1. Catalog of self-service AWS Resources (S3, RDS, Secrets, IRSA Role)
2. Sensible Terraform Module Boundaries:
    * Encryption at rest
    * Versioning
3. AWS Permission Boundaries
    * Limits on what can be created
    * Required Tags enforces at AWS level
    * Approved regions only
4. Runtime Governance
    * Pod Security standards enforced
    * Resource quotas for namespace
    * Mandatory network policies
    * Only approved container registries
5. K8s items
    * `Deployment` with resource limits
    * `Service` for load balancing
    * `Ingress` with TLS termination
    * `HPA` for auto-scaling
    * `NetworkPolicy` for security
    * `ServiceMonitor` for metrics
6. CI/CD Pipeline items
    * Docker image building
    * Security scanning
    * Helm deployment
    * Environment promotion
    * Rollback strategies
7. Network and Load Balancing items
    * Service discovery
    * Load Balancing
    * TLS termination
    * Network Security

### Permissions of Application Engineers (like Sarah)

Pipeline Service Account provides the following access and restrictions to App Engineers

* AWS
    * Create resources via approved Terraform modules only
    * Must use platform-managed state bucket
    * Must apply platform-required tags
    * Cannot create IAM policies (only use approved ones)
* K8s
    * Deploy to specific namespaces
    * Use approved Helm charts only
    * Cannot modify cluster-wide resources

### Responsibility and Permissions of the Platform Team

* Access 
    * Full AWS admin access with MFA and Breakglass account
    * Permission to edit the `golden-path-app-template` and `infrastructure-platform-devops` repositories
* Responsibilities
    * Terraform modules and their default configurations and limits on App Engineers
    * JSON schemas for infrastructure requests
    * Pipeline service account permissions
    * K8s cluster policies


### Platform Benefits

#### Simplifying Infrastructure Deployments and Compliance for App Developers

The Platform handles the following items:
* 200+ lines of Terraform code
* 15+ K8s resources
* Security policies & compliance
* Monitoring and alerting
* Cost optimization
* Disaster recovery
* Performance tuning

Her are some things Sarah doesn't need to know about to set up her app.

1. AWS Infrastructure Complexity
    * S3 bucket naming conventions
    * Bucket policies and IAM permissions
    * Encryption configuration (AES-256 vs KMS)
    * Versioning and lifecycle policies
    * Cross-region replication setup
    * Access logging configuration
    * Public access block settings
    * CORS configuration for web apps

2. Database Administration
    * Instance types (db.t3.micro vs db.r6g.large)
    * VPC and subnet group configuration
    * Security group rules and Network ACLs
    * Backup retention and maintenance windows
    * Multi-AZ deployment decisions
    * Read replica configuration
    * Performance monitoring setup
    * Database encryption keys
    * Connection pooling strategies

3. Security and IAM
    * IAM policy JSON syntax
    * OIDC provider configuration
    * Roles and policies
        * AWS service-linked roles
        * Resource-based vs. Identity based policies
    *  Service account annotations for IRSA
    * Token exchange mechanisms

4. K8s Orchestration
    * YAML syntax for K8s manifests
    * Resource requests and limits
    * Pod Disruption Budgets
    * HPA configuration
    * Ingress controller setup
    * TLS certificate management
    * Network policy rules
    * Service mesh integration (if used)
    * Pod security contexts
    * Affinity and anti-affinity rules

5. Secrets Management
    * AWS Secrets Manager and Parameter Store
    * Encryption in transit and at rest
    * Secret rotation strategy
    * External Secrets Operator configuration
    * K8s secret types
    * Base64 encoding/decoding
    * Secret mounting and environment variables
    * Cross-service secret sharing
    * Audit logging for secret access

6. Monitoring and Observability
    * Prometheus configuration and service discovery
    * Grafana dashboard creation
    * Alert manager setup
    * Log shipping and parsing
    * Metric naming conventions
    * SLI/SLO definition
    * Distributed tracing setup
    * Error rate and latency monitoring

7. CI/CD Pipeline Configuration
    * Jenkins pipeline syntax
    * Docker multi-stage builds
    * Container registry authentication
    * Helm chart templating
    * GitOps workflows
    * Pipeline security scanning tools
    * Artifact management

8. Networking and Load Balancing
    * VPC & Subnet design
    * Application Load Balancer configuration
    * Target and group health checks
    * SSL/TLS certificate management
    * DNS and Route53 setup
    * Network security groups
    * WAF rules and rate limiting

9. Environment-Specific Configurations
    * Cost optimization strategies
    * High availability patterns
    * Disaster recovery planning
    * Backup and restore procedures
    * Performance tuning per environment
    * Compliance requirements per environment

#### Multiple Layers of Governance
1. Infra request schema validation
2. Terraform Module Boundaries
3. AWS Permission Boundaries
4. Runtime Governance

#### Strategic Goals Supported by the Platform

* Reduced learning curve: Weeks of training -> Hours of onboarding
* Faster time to market: Focus on features, not infrastructure
* Better Security: Platform enforces best practices automatically
* Cost Optimization: Right-sized resources, no over-provisioning
* Risk Reduction: Standardized, tested patterns
