# Dreamboard Maintenance Guide

Routine operations and best practices for maintaining Dreamboard in production.

## Daily Operations

### Health Check
```bash
# Check services are running
gcloud run services list --region=southamerica-east1 --project=radiant-tide-401723

# Test backend
BACKEND_URL=$(gcloud run services describe dreamboard-backend \
  --region=southamerica-east1 --project=radiant-tide-401723 --format='value(status.url)')
curl $BACKEND_URL/health

# Check monitoring dashboard
# https://console.cloud.google.com/monitoring/dashboards?project=radiant-tide-401723
```

### Review Alerts
```bash
# Check if any alerts fired
gcloud alpha monitoring policies list --project=radiant-tide-401723

# View recent incidents
gcloud logging read "severity=ERROR" --limit=20 --project=radiant-tide-401723
```

### Check Costs
```bash
# View billing
gcloud billing accounts list
gcloud billing budgets list
```

## Weekly Operations

### Code Review
```bash
# Check recent changes
git log --oneline -10

# Review recent PRs
gh pr list --state closed --limit 5

# Check test coverage
npm run coverage  # frontend
pytest --cov      # backend
```

### Performance Review
```bash
# Check latency trends
# Access Cloud Monitoring dashboard

# Review error patterns
gcloud logging read "severity=ERROR" --limit=100 | grep -i pattern
```

### Dependency Updates
```bash
# Backend
cd backend
pip list --outdated

# Frontend
cd ../frontend/dreamboard
npm outdated
```

### Backup Verification
```bash
# Verify Terraform state is backed up
gsutil ls -l gs://radiant-tide-401723-tf-state/

# Verify Firestore exports
gsutil ls -r gs://BACKUP_BUCKET/exports/
```

## Monthly Operations

### Security Audit
```bash
# Review IAM permissions
gcloud projects get-iam-policy radiant-tide-401723 \
  --flatten="bindings[].members" \
  --format="table(bindings.role)"

# Check service account permissions
gcloud iam service-accounts get-iam-policy \
  dreamboard-account@radiant-tide-401723.iam.gserviceaccount.com
```

### Secret Rotation
```bash
# Rotate OAuth Client ID
# 1. Create new Client ID in Google Cloud Console
# 2. Update secret:
gcloud secrets versions add dreamboard-oauth-client-id \
  --data-file=- <<< "NEW_OAUTH_CLIENT_ID"

# 3. Cloud Run auto-picks up latest
# 4. Verify old one is no longer used

# 5. Delete old Client ID from GCP
```

### Backup Procedures
```bash
# Firestore backup
gcloud firestore export gs://BACKUP_BUCKET/exports/export-$(date +%Y%m%d)

# Terraform state backup (automatic in GCS)
gsutil ls -r gs://radiant-tide-401723-tf-state/
```

### Cost Analysis
```bash
# Generate billing report
gcloud billing accounts list
# Check trends in Cloud Console: Billing → Reports
```

### Update Dependencies
```bash
# Backend
cd backend
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt --upgrade
git add requirements.txt && git commit -m "chore: update Python dependencies"

# Frontend
cd ../frontend/dreamboard
npm update
npm audit fix
git add package-lock.json && git commit -m "chore: update npm dependencies"

# Push and create PR
git push origin feature/update-dependencies
gh pr create --title "chore: update dependencies" --body "Monthly dependency updates"
```

## Quarterly Operations

### Performance Optimization
```bash
# Review metrics
# - Check p95 latency trends
# - Check error rate trends
# - Check resource usage

# Optimize if needed:
# - Adjust Cloud Run memory/CPU
# - Optimize database queries
# - Add caching

# Update terraform:
# gke/terraform/cloud-run.tf
# resources {
#   limits = {
#     cpu    = "4"
#     memory = "16Gi"
#   }
# }
```

### Security Update
```bash
# Check for security updates
# - Docker base image updates
# - Framework security releases
# - Critical CVEs

# Update and test
docker build -t dreamboard-backend:security-update .
gcloud builds submit --tag=IMAGE_URL .

# Deploy via normal process
git push origin main
```

### Disaster Recovery Drill
```bash
# Test recovery procedures
# 1. Verify backups are restorable
gsutil cp gs://BACKUP_BUCKET/exports/export-latest . && \
  gcloud firestore import gs://BACKUP_BUCKET/exports/export-latest

# 2. Test Terraform rollback
cd gke/terraform
terraform plan  # Should show no changes if healthy

# 3. Verify secrets are accessible
gcloud secrets versions list dreamboard-oauth-client-id
```

## Annual Operations

### Audit & Compliance
```bash
# Full security audit
gcloud audit-logs

# Review all IAM bindings
gcloud projects get-iam-policy radiant-tide-401723 > iam-audit.json

# Check service accounts
gcloud iam service-accounts list

# Review Cloud Logging retention
gcloud logging sinks list
```

### Cost Optimization
```bash
# Analyze spending patterns
# Check Cloud Console: Billing → Reports

# Identify cost reduction opportunities:
# - Unused resources
# - Right-sizing
# - Reserved commitments
```

### Architecture Review
```bash
# Review current architecture
# - Still using Cloud Run?
# - Should we migrate to GKE?
# - Cost vs. performance tradeoff?

# Document decisions
# Update deployment docs
```

## Incident Response

### When Service is Down

**Step 1: Triage**
```bash
# Check service status
gcloud run services describe dreamboard-backend \
  --region=southamerica-east1 --project=radiant-tide-401723

# Check logs for errors
gcloud logging read "resource.labels.service_name=dreamboard-backend" \
  --limit=50 --format=json
```

**Step 2: Investigate**
```bash
# Common issues:
# 1. Out of memory: increase memory in cloud-run.tf
# 2. Database connection: check Firestore status
# 3. GCS permissions: check IAM bindings
# 4. Secret access: verify Secret Manager
# 5. High traffic: check autoscaling

# Check specific resource
gcloud run services describe dreamboard-backend --region=southamerica-east1 \
  --format='value(status.conditions)'
```

**Step 3: Fix**
```bash
# Option A: Rollback to previous revision
gcloud run services update-traffic dreamboard-backend \
  --to-revisions=PREVIOUS_REVISION=100 \
  --region=southamerica-east1

# Option B: Redeploy
cd gke/terraform
terraform apply -auto-approve

# Option C: Scale up if needed
# Edit cloud-run.tf and increase maxScale
terraform apply -auto-approve
```

**Step 4: Document**
```bash
# Create incident report
# - What happened
# - Root cause
# - Resolution
# - Prevention for future
```

## Monitoring Checklist

**Daily:**
- [ ] Services are running
- [ ] No critical alerts
- [ ] Logs are clean

**Weekly:**
- [ ] Error rate < 1%
- [ ] Latency p95 < 3s
- [ ] Memory usage < 80%
- [ ] Test backups

**Monthly:**
- [ ] Security audit
- [ ] Dependency updates
- [ ] Rotate secrets
- [ ] Cost analysis

**Quarterly:**
- [ ] Performance optimization
- [ ] Security updates
- [ ] Disaster recovery drill

**Annually:**
- [ ] Full audit
- [ ] Cost optimization
- [ ] Architecture review

## Useful Commands Reference

```bash
# Quick status check
gcloud run services list --region=southamerica-east1

# Tail logs
gcloud logging read "resource.type=cloud_run_revision" --stream

# Deploy from Terraform
cd gke/terraform && terraform apply -auto-approve

# Rollback
gcloud run services update-traffic dreamboard-backend \
  --to-revisions=REVISION_NAME=100

# SSH into container (debug)
gcloud run services describe dreamboard-backend --region=southamerica-east1

# Update env variable
gcloud run services update dreamboard-backend \
  --update-env-vars KEY=VALUE \
  --region=southamerica-east1

# Scale up
gcloud run services update dreamboard-backend \
  --max-instances=100 \
  --region=southamerica-east1
```

## Escalation Path

**Tier 1: Automated Alerts**
- Email notifications for critical issues
- Check Cloud Monitoring dashboard

**Tier 2: Manual Investigation**
- Review logs and metrics
- Check service status
- Test endpoints

**Tier 3: Senior Review**
- Complex issues
- Architecture changes
- Security concerns

**Contact:**
- Gustavo Hervas: gustavo.hervas@monks.com
- On-call rotation: [Configure in alerting]

## References

- [Cloud Run Monitoring](https://cloud.google.com/run/docs/monitoring)
- [Cloud Logging](https://cloud.google.com/logging/docs)
- [Firestore Backups](https://cloud.google.com/firestore/docs/manage-data/export-import)
- [GCP Best Practices](https://cloud.google.com/docs/solutions/gke-best-practices)

---

**Last Updated:** 2026-09-28  
**Version:** 1.0
