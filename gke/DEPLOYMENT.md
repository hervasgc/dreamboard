# Dreamboard Deployment Guide

Complete guide for deploying and maintaining Dreamboard on Google Cloud Run with Infrastructure as Code.

**Status:** ✅ Fully Automated with GitHub Actions

## Quick Start

### Prerequisites
```bash
# Required tools
gcloud auth login --update-adc
gcloud config set project radiant-tide-401723
terraform version  # >= 1.0
gh auth login      # GitHub CLI
```

### 5-Minute Deploy
```bash
# 1. Setup (one-time)
cd gke/terraform/
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values

# 2. Initialize
terraform init -backend-config="bucket=radiant-tide-401723-tf-state"

# 3. Deploy
terraform plan && terraform apply
```

## Full Deployment Workflow

### Phase 1: Infrastructure (Terraform)

**Setup Terraform state:**
```bash
# Create state bucket (one-time)
gsutil mb -p radiant-tide-401723 -l southamerica-east1 gs://radiant-tide-401723-tf-state
gsutil versioning set on gs://radiant-tide-401723-tf-state
```

**Configure variables:**
```bash
cp gke/terraform/terraform.tfvars.example gke/terraform/terraform.tfvars

# Edit with your values:
# - project_id: radiant-tide-401723
# - region: southamerica-east1
# - cloudrun_image_backend: Artifact Registry URL
# - cloudrun_image_frontend: Artifact Registry URL
# - oauth_client_id: Your OAuth ID
# - alert_email: Your email for alerts
```

**Deploy infrastructure:**
```bash
cd gke/terraform/
terraform init
terraform validate
terraform plan
terraform apply
```

**Get service URLs:**
```bash
terraform output backend_url
terraform output frontend_url
```

### Phase 2: CI/CD (GitHub Actions)

**Setup Workload Identity Federation:**
See `gke/PHASE-2-SETUP.md` for complete instructions.

**Add GitHub Secrets:**
```
GCP_PROJECT_ID          = radiant-tide-401723
WIF_PROVIDER            = projects/{number}/locations/global/workloadIdentityPools/github-actions-pool/providers/github-provider
GCP_SERVICE_ACCOUNT     = dreamboard-github-actions@radiant-tide-401723.iam.gserviceaccount.com
REGISTRY                = southamerica-east1-docker.pkg.dev/radiant-tide-401723/dreamboard-docker-repo
TF_STATE_BUCKET         = radiant-tide-401723-tf-state
OAUTH_CLIENT_ID         = 654852193118-...
```

**Workflows active:**
- `.github/workflows/build-images.yml` → Build Docker images
- `.github/workflows/deploy.yml` → Deploy with Terraform
- `.github/workflows/validate-pr.yml` → Validate PRs

### Phase 3: Secrets & Monitoring (Optional but Recommended)

**Setup secrets:**
```bash
# Populate secrets
gcloud secrets versions add dreamboard-oauth-client-id \
  --data-file=- <<< "YOUR_OAUTH_CLIENT_ID"

# Verify
gcloud secrets versions access latest --secret=dreamboard-oauth-client-id
```

**Access monitoring dashboard:**
```
https://console.cloud.google.com/monitoring/dashboards?project=radiant-tide-401723
```

## Deployment Models

### Model 1: Manual (Development)
```bash
# Build images locally
cd backend && gcloud builds submit --tag=IMAGE_URL .

# Deploy via Terraform
cd gke/terraform && terraform apply -auto-approve
```

**Best for:** Local testing, quick iterations

### Model 2: Automated (Recommended)
```bash
# Push code to main
git push origin main

# GitHub Actions automatically:
# 1. Builds images
# 2. Validates PRs
# 3. Deploys via Terraform
```

**Best for:** Production, team collaboration

### Model 3: CI/CD Only
```bash
# GitHub Actions builds images only
# Manual terraform deploy
cd gke/terraform && terraform apply -auto-approve
```

**Best for:** Hybrid workflows

## File Structure

```
gke/
├── DEPLOYMENT.md              # This file
├── PHASE-1-SETUP.md          # Terraform setup (Phase 1)
├── PHASE-2-SETUP.md          # CI/CD setup (Phase 2)
├── PHASE-3-SETUP.md          # Secrets & Monitoring (Phase 3)
├── MAINTENANCE.md            # Ops & maintenance
├── TROUBLESHOOTING.md        # Common issues
└── terraform/
    ├── main.tf               # Main configuration
    ├── cluster.tf            # GKE configuration (if using)
    ├── cloud-run.tf          # Cloud Run services
    ├── secrets.tf            # Secret Manager
    ├── monitoring.tf         # Cloud Monitoring
    ├── iam.tf                # IAM roles
    ├── variables.tf          # Variables
    ├── outputs.tf            # Outputs
    ├── versions.tf           # Provider versions
    ├── backend.tf            # State backend
    ├── terraform.tfvars.example
    ├── SECRETS-INTEGRATION.md
    └── .gitignore
```

## Deployment Checklist

### Before First Deploy
- [ ] gcloud CLI installed and authenticated
- [ ] Terraform installed (>= 1.0)
- [ ] Google OAuth Client ID created
- [ ] Terraform state bucket created
- [ ] terraform.tfvars configured
- [ ] Alert email configured

### Before Each Deploy
- [ ] All tests passing
- [ ] Code reviewed and approved
- [ ] Docker images built successfully
- [ ] No uncommitted changes

### After Deploy
- [ ] Backend service healthy (check logs)
- [ ] Frontend loads without errors
- [ ] OAuth login works
- [ ] Monitoring dashboard shows metrics
- [ ] No alerts triggered

## Common Operations

### View Service Status
```bash
gcloud run services list --region=southamerica-east1 --project=radiant-tide-401723
```

### View Service Logs
```bash
# Backend
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=dreamboard-backend" \
  --limit=50 --format=json

# Frontend
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=dreamboard-frontend" \
  --limit=50 --format=json
```

### Scale Services
Cloud Run auto-scales. To adjust limits:

```hcl
# In gke/terraform/cloud-run.tf, update:
annotations = {
  "autoscaling.knative.dev/maxScale" = "100"  # Max concurrent requests
  "autoscaling.knative.dev/minScale" = "1"    # Min idle instances
}
```

Then: `terraform apply -auto-approve`

### Update Environment Variables
```hcl
# Edit gke/terraform/cloud-run.tf
env {
  name  = "VARIABLE_NAME"
  value = "new_value"
}

# Deploy
terraform apply -auto-approve
```

### Rollback to Previous Version
```bash
# Cloud Run keeps revisions. Route traffic to previous:
gcloud run services update-traffic dreamboard-backend \
  --to-revisions=PREVIOUS_REVISION=100 \
  --region=southamerica-east1 \
  --project=radiant-tide-401723
```

### Update Secrets
```bash
# Add new version
gcloud secrets versions add dreamboard-oauth-client-id \
  --data-file=- <<< "NEW_VALUE"

# Cloud Run automatically picks up latest
# Or redeploy:
terraform apply -auto-approve
```

## Monitoring & Alerts

### View Dashboard
```
https://console.cloud.google.com/monitoring/dashboards?project=radiant-tide-401723
```

### Configure Alerts
Alerts send email when:
- Error rate > 1% (5 min)
- Latency p95 > 3s (5 min)
- Service down (3 min)
- Memory > 80% (5 min)

Edit thresholds in `gke/terraform/monitoring.tf`

### View Logs
```bash
# All errors
gcloud logging read "severity=ERROR" --limit=50

# Specific service
gcloud logging read "resource.labels.service_name=dreamboard-backend" --limit=50

# Stream live
gcloud logging read ... --stream
```

## Cost Monitoring

### Estimate Costs
```bash
# Cloud Run: ~$0.40/month (light usage)
# Cloud Build: ~$0.003/min (included in free tier)
# Cloud Storage: ~$0.02/month (tiny bucket)
# Monitoring: Free

# Total: ~$5-20/month depending on traffic
```

### Set Budget Alerts
```bash
# In Google Cloud Console:
# Billing → Budgets and alerts
# Set budget: $50/month
# Alert at 50%, 90%, 100%
```

## Security Checklist

- [ ] OAuth Client ID in Secret Manager
- [ ] Service account with minimal permissions
- [ ] Workload Identity Federation (no stored keys)
- [ ] Cloud Run authentication required
- [ ] VPC Service Controls (optional)
- [ ] Cloud Armor DDoS protection (optional)
- [ ] TLS/HTTPS enforced

## Backup & Recovery

### Backup Terraform State
```bash
# State is versioned in GCS
gsutil ls -r gs://radiant-tide-401723-tf-state/

# Download a version
gsutil cp gs://radiant-tide-401723-tf-state/terraform-state .
```

### Backup Firestore
```bash
# Schedule export
gcloud firestore export gs://BACKUP_BUCKET/exports/export-$(date +%Y%m%d)
```

### Backup GCS Data
```bash
# Enable versioning
gsutil versioning set on gs://monks-dreamboard
```

## Troubleshooting

See `gke/TROUBLESHOOTING.md` for common issues and solutions.

## Maintenance

See `gke/MAINTENANCE.md` for routine operations and best practices.

## Support & References

- [Terraform Google Cloud Provider](https://registry.terraform.io/providers/hashicorp/google/latest)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Dreamboard Backend README](../../backend/README.md)
- [Dreamboard Frontend README](../../frontend/README.md)

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2026-09-28 | 1.0 | Initial: Phases 1-3 (IaC, CI/CD, Secrets) |

---

**Last Updated:** 2026-09-28  
**Maintained By:** Claude Haiku + Gustavo Hervas  
**Status:** Production Ready
