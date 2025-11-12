# ArgoCD GitOps Repository for AWS EKS Automode

Production-ready GitOps repository for managing Kubernetes workloads on AWS EKS Automode using ArgoCD with the App of Apps pattern.

## 🎯 Overview

This repository implements a scalable GitOps structure for deploying applications across multiple environments using:
- **ArgoCD** - GitOps continuous delivery
- **Helm** - Package management with separate charts per workload
- **ApplicationSets** - Auto-discovery of workloads with Git directory generator
- **Multi-Source Applications** - Helm charts + external ConfigMaps
- **AWS EKS Automode** - Automated cluster infrastructure

## 🏗️ Architecture

### App of Apps Pattern
```
root-app (Bootstrap)
├── platform-apps (Sync Wave 1)
│   ├── EBS StorageClass
│   ├── EFS StorageClass
│   └── Node Pools
├── environment-namespaces (Sync Wave 2)
│   ├── cldev01 namespace + quota
│   ├── cldev02 namespace + quota
│   └── clat01 namespace + quota
└── environment-applicationsets (Sync Wave 3)
    ├── cldev01-workloads ApplicationSet
    ├── cldev02-workloads ApplicationSet
    └── clat01-workloads ApplicationSet
        └── Auto-generates Applications for each chart in charts/*
            ├── app1-{env}
            ├── app2-{env}
            └── app3-{env}
```

### Multi-Source Applications
Each workload Application uses two sources:
1. **Helm Chart**: `charts/{app}/` - Infrastructure and deployment manifests
2. **ConfigMaps**: `environments/{env}/config/{app}/` - Pipeline-generated configuration

## 📁 Repository Structure

```
argocd-pilot/
│
├── argocd/                        # ArgoCD Applications
│   ├── root-app.yaml              # 🚀 START HERE - Bootstrap everything
│   └── platform-apps.yaml         # Platform infrastructure
│
├── platform/                      # Cluster-wide resources (Sync Wave 1)
│   ├── storage/
│   │   ├── ebs-storageclass.yaml  # EBS gp3 storage (default)
│   │   └── efs-storageclass.yaml  # EFS storage (ReadWriteMany)
│   ├── node-pools/
│   │   ├── general-purpose.yaml   # m6i instances, no taints
│   │   └── compute-optimized.yaml # c6i instances, tainted
│   └── cluster-config/
│       ├── resource-quotas.yaml   # Cluster resource limits
│       └── limit-ranges.yaml      # Default pod limits
│
├── charts/                        # Helm charts (one per workload)
│   ├── app1/                      # Standard web application
│   │   ├── Chart.yaml
│   │   ├── values.yaml            # Default values for all environments
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── ingress.yaml       # AWS ALB ingress
│   │       ├── hpa.yaml
│   │       ├── pdb.yaml
│   │       ├── configmap.yaml
│   │       ├── secret.yaml
│   │       ├── serviceaccount.yaml # IRSA support
│   │       ├── pvc.yaml
│   │       ├── networkpolicy.yaml
│   │       └── _helpers.tpl
│   ├── app2/                      # Compute-intensive workload
│   ├── app3/                      # Ingress-focused (API Gateway)
│   └── workload/                  # Template for creating new apps
│
├── environments/                  # Environment configurations
│   ├── CONFIG-README.md           # External ConfigMap documentation
│   │
│   ├── cldev01/                   # Development Environment 01
│   │   ├── namespace.yaml         # Namespace: cldev01
│   │   ├── resource-quota.yaml    # 20 CPU, 40Gi memory
│   │   ├── applicationset.yaml    # Auto-generates app1/app2/app3 Applications
│   │   │
│   │   ├── config/                # Pipeline-generated ConfigMaps
│   │   │   ├── app1/
│   │   │   │   └── application-config.yaml
│   │   │   ├── app2/
│   │   │   │   └── application-config.yaml
│   │   │   └── app3/
│   │   │       └── application-config.yaml
│   │   │
│   │   └── values/                # Helm value overrides (minimal)
│   │       ├── app1-values.yaml   # Only values that differ from chart defaults
│   │       ├── app2-values.yaml
│   │       └── app3-values.yaml
│   │
│   ├── cldev02/                   # Development Environment 02
│   │   └── (same structure)
│   │
│   └── clat01/                    # Acceptance Test (Production-like)
│       └── (same structure)
│
├── .github/workflows/             # CI/CD Pipelines
│   ├── validate.yml               # YAML lint, Helm lint, kubeconform, Trivy
│   └── argocd-sync.yml            # Trigger ArgoCD sync on push
│
├── .gitignore
├── .yamllint.yml
├── README.md                      # This file
├── EXTERNAL-CONFIGMAPS.md         # Pipeline ConfigMap integration guide
├── QUICKREF.md                    # Quick reference
├── EXTENDING.md                   # How to add apps/environments
├── CHANGES.md                     # Architecture decisions
└── EXAMPLE-compute-workload-values.yaml
```

## 🚀 Quick Start

### Prerequisites
- EKS cluster with ArgoCD installed
- kubectl configured with cluster access
- ArgoCD CLI (optional, for easier management)

### 1. Clone Repository
```bash
git clone https://github.com/your-org/argocd-pilot.git
cd argocd-pilot
```

### 2. Update Configuration
```bash
# Update repository URLs
find . -name "*.yaml" -type f -exec sed -i 's|your-org/argocd-pilot|YOUR_ORG/YOUR_REPO|g' {} +

# Update AWS account ID
find environments -name "*-values.yaml" -type f -exec sed -i 's|123456789012|YOUR_AWS_ACCOUNT|g' {} +

# Update EFS filesystem ID (if using EFS)
sed -i 's|fs-xxxxxxxxx|YOUR_EFS_ID|g' platform/storage/efs-storageclass.yaml
```

### 3. Deploy Root Application
```bash
# This bootstraps everything
kubectl apply -f argocd/root-app.yaml

# Watch the deployment
kubectl get applications -n argocd -w
```

### 4. Verify Deployment
```bash
# Check platform resources
kubectl get storageclass
kubectl get resourcequota -A

# Check ApplicationSets (should show 3)
kubectl get applicationsets -n argocd

# Check auto-generated Applications (should show 9: 3 apps × 3 environments)
kubectl get applications -n argocd

# Check workload pods
kubectl get pods -n cldev01
kubectl get pods -n cldev02
kubectl get pods -n clat01
```

## 📊 Environments

| Environment | Namespace | Purpose | Resources | Replicas | Autoscaling |
|------------|-----------|---------|-----------|----------|-------------|
| **cldev01** | cldev01 | Development 01 | 20 CPU, 40Gi | 1 | Disabled |
| **cldev02** | cldev02 | Development 02 | 20 CPU, 40Gi | 1 | Disabled |
| **clat01** | clat01 | Acceptance Test | 50 CPU, 100Gi | 2-3 | Enabled |

## 📦 Workloads

| Application | Type | Features | Node Pool |
|------------|------|----------|-----------|
| **app1** | Web Application | Standard deployment, ingress, HPA | General Purpose |
| **app2** | Worker/Compute | Higher CPU, persistent storage | Compute Optimized |
| **app3** | API Gateway | Advanced ingress, multi-domain, WAF | General Purpose |

## 🔧 Configuration Management

### Helm Values Strategy
**Only override what differs from chart defaults.**

```
Final Values = Chart Default + Environment Override
                (charts/app1/values.yaml) + (environments/cldev01/values/app1-values.yaml)
```

#### Chart Defaults (`charts/app1/values.yaml`)
- Apply to ALL environments
- Sensible defaults for production
- Infrastructure/deployment patterns
- ~150 lines

#### Environment Overrides (`environments/{env}/values/app1-values.yaml`)
- ONLY values that differ from defaults
- Environment-specific: image tags, replicas, resources
- ~30-150 lines depending on complexity

**Example:**
```yaml
# charts/app1/values.yaml (default)
replicaCount: 1
image:
  repository: nginx
  tag: latest
resources:
  limits:
    cpu: 500m
    memory: 512Mi

# environments/cldev01/values/app1-values.yaml (override only what's different)
image:
  repository: your-registry.dkr.ecr.us-east-1.amazonaws.com/app1
  pullPolicy: Always
  tag: dev-latest
```

### External ConfigMaps (Pipeline-Generated)

Your DevOps pipeline can generate application-specific ConfigMaps:

```yaml
# Pipeline generates and commits this file
# environments/cldev01/config/app1/application-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-application-config
  namespace: cldev01
  labels:
    managed-by: devops-pipeline
  annotations:
    pipeline.run: "BUILD-12345"
data:
  application.yaml: |
    database:
      host: dev-db.cldev01.svc.cluster.local
    features:
      newUI: false
```

ArgoCD automatically applies it with the Helm chart. See `EXTERNAL-CONFIGMAPS.md` for details.

## 🎨 Adding New Resources

### Add a New Workload

1. **Create Helm Chart**
   ```bash
   cp -r charts/workload charts/app4
   cd charts/app4
   # Edit Chart.yaml, values.yaml, templates/
   # Update all template helpers: workload. → app4.
   ```

2. **Create Environment Values** (minimal overrides only)
   ```bash
   # environments/cldev01/values/app4-values.yaml
   image:
     repository: your-registry/app4
     tag: dev-latest
   ingress:
     enabled: true
     hosts:
       - host: app4.cldev01.example.com
   ```

3. **Create ConfigMap Placeholder** (optional)
   ```bash
   # environments/cldev01/config/app4/application-config.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: app4-application-config
     namespace: cldev01
   data:
     config.yaml: |
       # Pipeline will generate this
   ```

4. **Commit and Push**
   ```bash
   git add charts/app4 environments/*/values/app4-values.yaml environments/*/config/app4/
   git commit -m "Add app4 workload"
   git push
   ```

5. **ApplicationSet Auto-Discovers**
   - No changes needed to ApplicationSet
   - Scans `charts/*` directory
   - Creates `app4-cldev01`, `app4-cldev02`, `app4-clat01` Applications
   - Deploys within 3 minutes (or immediately with webhook)

### Add a New Environment

1. **Create Environment Directory**
   ```bash
   mkdir -p environments/clprod01/{config/{app1,app2,app3},values}
   ```

2. **Create Namespace and Quota**
   ```yaml
   # environments/clprod01/namespace.yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: clprod01
   ```

3. **Create ApplicationSet**
   ```bash
   cp environments/cldev01/applicationset.yaml environments/clprod01/
   # Update all instances of cldev01 → clprod01
   ```

4. **Create Values Files** (one per workload)
   ```bash
   cp environments/clat01/values/app1-values.yaml environments/clprod01/values/
   # Edit for production settings
   ```

5. **Commit and Push**
   ```bash
   git add environments/clprod01
   git commit -m "Add clprod01 environment"
   git push
   ```
   
   Root app auto-discovers the new environment (scans `environments/*/namespace.yaml`)

## 🔍 Monitoring and Troubleshooting

### View Application Status
```bash
# List all Applications
kubectl get applications -n argocd

# View specific Application
argocd app get app1-cldev01

# View sync status
argocd app sync app1-cldev01 --dry-run

# Manual sync
argocd app sync app1-cldev01
```

### Common Issues

#### Application Not Auto-Created
```bash
# Check ApplicationSet
kubectl get applicationset -n argocd
kubectl describe applicationset cldev01-workloads -n argocd

# Verify chart exists
ls -la charts/app1

# Check values file exists
ls -la environments/cldev01/values/app1-values.yaml
```

#### Sync Failed
```bash
# View sync errors
argocd app get app1-cldev01

# Check pod events
kubectl describe pod -n cldev01 -l app.kubernetes.io/name=app1

# View logs
kubectl logs -n cldev01 -l app.kubernetes.io/name=app1
```

#### ConfigMap Not Applied
```bash
# Check if ConfigMap exists in Git
ls -la environments/cldev01/config/app1/

# Verify multi-source setup
kubectl get application app1-cldev01 -n argocd -o yaml | grep -A 10 sources

# Check ConfigMap in cluster
kubectl get configmap -n cldev01 | grep application-config
```

#### Wrong Values Applied
```bash
# Test Helm template rendering
helm template app1 charts/app1/ \
  -f environments/cldev01/values/app1-values.yaml \
  --debug

# Check final merged values
helm template app1 charts/app1/ \
  -f environments/cldev01/values/app1-values.yaml \
  --debug 2>&1 | grep -A 999 "COMPUTED VALUES"
```

## 🔐 Security Best Practices

### Secrets Management
**Never store secrets in Git!**

Use one of:
- **AWS Secrets Manager** + External Secrets Operator
- **AWS Systems Manager Parameter Store** + External Secrets Operator
- **HashiCorp Vault**
- Kubernetes Secrets (encrypted at rest in etcd)

```yaml
# ❌ DON'T do this
data:
  password: "plaintext-password"

# ✅ DO this - Use External Secrets
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app1-secrets
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: app1-secrets
  data:
    - secretKey: database-password
      remoteRef:
        key: /prod/app1/db-password
```

### IRSA (IAM Roles for Service Accounts)
Every workload uses IRSA for AWS access:

```yaml
# In environment values
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/app1-cldev01-role
```

### Network Policies
Enable network policies in production:

```yaml
networkPolicy:
  enabled: true
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
```

## 📈 Scaling and High Availability

### Horizontal Pod Autoscaling
```yaml
# In environment values (clat01)
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### Pod Disruption Budgets
```yaml
# Ensure availability during updates
podDisruptionBudget:
  enabled: true
  minAvailable: 1  # At least 1 pod during rolling updates
```

### Pod Anti-Affinity
```yaml
# Spread pods across nodes and AZs
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
      - weight: 50
        podAffinityTerm:
          topologyKey: topology.kubernetes.io/zone
```

## 🏷️ Node Selectors and Tolerations

### General Purpose Nodes (Default)
```yaml
nodeSelector:
  workload-type: general
# No tolerations needed
```

### Compute-Optimized Nodes
```yaml
nodeSelector:
  workload-type: compute-intensive

tolerations:
  - key: "workload-type"
    operator: "Equal"
    value: "compute-intensive"
    effect: "NoSchedule"
```

## 🌐 Ingress and ALB

### Basic HTTP Ingress (Development)
```yaml
ingress:
  enabled: true
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
  hosts:
    - host: app1.cldev01.example.com
```

### HTTPS with ACM Certificate (Production)
```yaml
ingress:
  enabled: true
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
  hosts:
    - host: app1.clat01.example.com
```

### WAF Protection
```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:xxx:regional/webacl/xxx
```

## 💾 Storage

### EBS (Block Storage - Default)
```yaml
persistence:
  enabled: true
  storageClass: ebs-gp3  # Default, encrypted, 3000 IOPS
  size: 10Gi
  accessMode: ReadWriteOnce
```

### EFS (Shared Storage)
```yaml
persistence:
  enabled: true
  storageClass: efs
  size: 100Gi
  accessMode: ReadWriteMany  # Multiple pods can share
```

## 🔄 CI/CD Integration

### Validation Workflow
Runs on every PR:
- YAML linting (yamllint)
- Helm chart linting
- Kubernetes manifest validation (kubeconform)
- Security scanning (Trivy)

### ArgoCD Sync Workflow
Runs on push to main:
- Triggers ArgoCD sync for all Applications
- Can be environment-specific with path filters

### Pipeline ConfigMap Generation
```yaml
# Example GitHub Actions workflow
- name: Generate ConfigMap
  run: |
    cat > environments/cldev01/config/app1/application-config.yaml <<EOF
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app1-application-config
      namespace: cldev01
      annotations:
        build.id: "${{ github.run_number }}"
    data:
      application.yaml: |
        version: "${{ github.sha }}"
        environment: cldev01
    EOF
    
- name: Commit ConfigMap
  run: |
    git add environments/
    git commit -m "Update app1 config - Build ${{ github.run_number }}"
    git push
```

## 📚 Additional Documentation

- **EXTERNAL-CONFIGMAPS.md** - Pipeline-generated ConfigMap integration
- **QUICKREF.md** - Quick reference for common operations
- **EXTENDING.md** - Detailed guide for adding apps/environments
- **CHANGES.md** - Architecture decisions and changelog
- **environments/CONFIG-README.md** - ConfigMap detailed documentation

## 🤝 Contributing

1. Create feature branch
2. Make changes
3. Test with `helm template` and `kubectl apply --dry-run`
4. Run validation: `yamllint .` and `helm lint charts/*`
5. Create PR

## 📝 Customization Checklist

Before deploying to your cluster:

- [ ] Update repository URLs in all `*.yaml` files
- [ ] Update AWS account ID in all environment values files
- [ ] Update EFS filesystem ID in `platform/storage/efs-storageclass.yaml`
- [ ] Update IAM role ARNs in environment values files
- [ ] Update ECR registry URLs in environment values files
- [ ] Update ingress hostnames
- [ ] Update ACM certificate ARNs (for HTTPS)
- [ ] Configure WAF WebACL ARNs (if using WAF)
- [ ] Review and adjust resource quotas per environment
- [ ] Review and adjust pod resource limits/requests
- [ ] Configure ArgoCD notifications (Slack, email)
- [ ] Set up External Secrets Operator (for production)
- [ ] Configure monitoring (Prometheus, Grafana)
- [ ] Set up logging aggregation (CloudWatch, ELK)

## 🎓 Key Concepts

### GitOps
- Git is the source of truth
- Declarative infrastructure
- Automated synchronization
- Audit trail via Git history

### App of Apps Pattern
- Root app manages other apps
- Hierarchical structure
- Sync waves for ordering
- Single entry point

### ApplicationSets
- Auto-generate Applications
- Git directory generator
- Reduces manual Application creation
- Scales to hundreds of apps

### Multi-Source Applications
- Combine multiple sources
- Helm chart + external configs
- Separate concerns
- Flexible deployment patterns

### Minimal Value Overrides
- Chart defaults for common settings
- Override only differences
- DRY principle
- Easier maintenance

## 📞 Support

For issues and questions:
1. Check troubleshooting section above
2. Review ArgoCD logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`
3. Check Application status: `argocd app get <app-name>`
4. Review this README and additional documentation

## 📄 License

[Your License Here]

---

**Built with ❤️ for AWS EKS Automode + ArgoCD**
