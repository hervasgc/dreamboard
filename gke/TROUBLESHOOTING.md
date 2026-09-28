# Dreamboard Troubleshooting Guide

Solutions to common issues and error messages.

## Deployment Issues

### Error: "Requested entity was not found"

**Symptom:**
```
Error: Error creating service: googleapi: Error 404: Requested entity was not found., notFound
```

**Cause:** Docker image not found in Artifact Registry

**Solution:**
```bash
# Verify image exists
gcloud artifacts docker images list \
  --repository=dreamboard-docker-repo \
  --location=southamerica-east1

# If missing, build and push
cd backend
gcloud builds submit \
  --region=southamerica-east1 \
  --tag=southamerica-east1-docker.pkg.dev/radiant-tide-401723/dreamboard-docker-repo/dreamboard-backend:latest .
```

---

### Error: "Permission denied"

**Symptom:**
```
Error: googleapi: Error 403: Permission denied
```

**Cause:** Service account lacks necessary permissions

**Solution:**
```bash
# Check service account permissions
gcloud projects get-iam-policy radiant-tide-401723 \
  --flatten="bindings[].members" \
  --filter="bindings.members:dreamboard-account@*"

# Verify roles include:
# - roles/run.admin
# - roles/artifactregistry.reader
# - roles/storage.admin
# - roles/aiplatform.user
```

---

### Error: "State bucket not found"

**Symptom:**
```
Error: Error reading service.tf: googleapi: Error 403: Forbidden, forbidden
```

**Cause:** Terraform state bucket doesn't exist or backend not configured

**Solution:**
```bash
# Create bucket
gsutil mb -p radiant-tide-401723 -l southamerica-east1 \
  gs://radiant-tide-401723-tf-state

# Update gke/terraform/backend.tf
terraform {
  backend "gcs" {
    bucket = "radiant-tide-401723-tf-state"
  }
}

# Reinitialize
terraform init -backend-config="bucket=radiant-tide-401723-tf-state"
```

---

## Cloud Run Issues

### Backend Service Keeps Crashing

**Symptom:** Cloud Run revision keeps crashing, never reaches 100% traffic

**Check logs:**
```bash
gcloud logging read "resource.labels.service_name=dreamboard-backend" \
  --limit=50 --format=json | jq '.[] | .textPayload'
```

**Common causes:**

**1. Out of Memory (OOM)**
```bash
# Error: "Container terminated with exit code 137"
# Solution: Increase memory
# Edit gke/terraform/cloud-run.tf:
resources {
  limits = {
    cpu    = "4"
    memory = "32Gi"  # Increase from 16Gi
  }
}
terraform apply -auto-approve
```

**2. Environment Variable Missing**
```bash
# Error: "KeyError: 'VARIABLE_NAME'"
# Solution: Verify all required env vars are set
gcloud run services describe dreamboard-backend \
  --region=southamerica-east1 \
  --format='value(spec.template.spec.containers[0].env)'
```

**3. Database Connection Failed**
```bash
# Error: "Failed to connect to Firestore"
# Check Firestore is accessible:
gcloud firestore databases list

# Check IAM permissions include roles/datastore.user
```

**4. Secret Not Accessible**
```bash
# Error: "secret-backend-api-key: not found"
# Solution: Verify secret exists and IAM binding
gcloud secrets list
gcloud secrets get-iam-policy dreamboard-oauth-client-id
```

---

### High Latency (p95 > 3 seconds)

**Diagnosis:**
```bash
# Check monitoring dashboard
# https://console.cloud.google.com/monitoring/dashboards?project=radiant-tide-401723

# Check logs for slow requests
gcloud logging read "jsonPayload.latency > 3000" \
  --limit=20 --format=json
```

**Solutions:**

1. **Increase CPU/Memory:**
```hcl
resources {
  limits = {
    cpu    = "8"      # Increase from 4
    memory = "32Gi"   # Increase from 16Gi
  }
}
```

2. **Optimize Code:**
```bash
# Profile backend
python -m cProfile app/main.py

# Check slow database queries
# Add logging to long operations
```

3. **Add Caching:**
```python
# Cache expensive API calls
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_operation():
    return result
```

4. **Scale Out:**
```hcl
annotations = {
  "autoscaling.knative.dev/maxScale" = "1000"  # More instances
}
```

---

### High Error Rate (> 1%)

**Diagnosis:**
```bash
# Check error logs
gcloud logging read "severity=ERROR" \
  --limit=50 --format=json

# Count 5xx errors
gcloud logging read "httpRequest.status >= 500" \
  --limit=100 | wc -l
```

**Solutions:**

1. **Check Application Logs**
```bash
gcloud logging read "resource.labels.service_name=dreamboard-backend" \
  --limit=100 --format='json' | jq '.[] | select(.severity=="ERROR")'
```

2. **Specific Error Analysis**
```bash
# Check for specific error pattern
gcloud logging read "textPayload =~ /error message/" \
  --limit=20
```

3. **Rollback to Previous Version**
```bash
# List revisions
gcloud run revisions list --service=dreamboard-backend \
  --region=southamerica-east1

# Route to previous
gcloud run services update-traffic dreamboard-backend \
  --to-revisions=REVISION_NAME=100 \
  --region=southamerica-east1
```

---

## GitHub Actions Issues

### Workflow Fails: "No such object"

**Symptom:**
```
Error: Error reading bucket contents: googleapi: Error 404: Not Found, notFound
```

**Cause:** TF_STATE_BUCKET secret not configured or incorrect

**Solution:**
```bash
# Verify secret in GitHub
# Settings → Secrets → TF_STATE_BUCKET

# Should be: radiant-tide-401723-tf-state

# Verify bucket exists
gsutil ls gs://radiant-tide-401723-tf-state/

# If missing, create it
gsutil mb gs://radiant-tide-401723-tf-state
```

---

### Workflow Fails: "WIF authentication failed"

**Symptom:**
```
Error: google.auth.exceptions.DefaultCredentialsError: Could not automatically determine credentials
```

**Cause:** Workload Identity Federation not configured or repo name doesn't match

**Solution:**
```bash
# Verify WIF provider
gcloud iam workload-identity-pools providers describe github-provider \
  --location=global \
  --workload-identity-pool=github-actions-pool \
  --project=radiant-tide-401723

# Check repository matches
# Should be: hervasgc/dreamboard

# If wrong, recreate provider with correct repo
gcloud iam workload-identity-pools providers delete github-provider \
  --location=global \
  --workload-identity-pool=github-actions-pool

# Recreate with correct repo
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --attribute-mapping="repository=assertion.repository" \
  --issuer-uri=https://token.actions.githubusercontent.com
```

---

### Workflow Fails: "Service account not found"

**Symptom:**
```
Error: Unable to generate access token
```

**Cause:** GCP_SERVICE_ACCOUNT secret missing or incorrect

**Solution:**
```bash
# Verify service account exists
gcloud iam service-accounts list --filter="email:dreamboard-github-actions@*"

# Verify GitHub secret
# Settings → Secrets → GCP_SERVICE_ACCOUNT

# Should be: dreamboard-github-actions@radiant-tide-401723.iam.gserviceaccount.com

# If missing, create service account
gcloud iam service-accounts create dreamboard-github-actions \
  --display-name="GitHub Actions CI/CD"
```

---

## Secret Manager Issues

### "Secret access denied"

**Symptom:**
```
Error: Permission denied on resource 'projects/*/secrets/dreamboard-oauth-client-id'
```

**Cause:** Service account doesn't have `secretmanager.secretAccessor` role

**Solution:**
```bash
# Add role to service account
gcloud secrets add-iam-policy-binding dreamboard-oauth-client-id \
  --member="serviceAccount:dreamboard-account@radiant-tide-401723.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor" \
  --project=radiant-tide-401723

# Verify
gcloud secrets get-iam-policy dreamboard-oauth-client-id
```

---

### "Secret version not found"

**Symptom:**
```
Error: secrets.googleapis.com/not_found: Secret version not found
```

**Cause:** Secret has no versions or version deleted

**Solution:**
```bash
# List secret versions
gcloud secrets versions list dreamboard-oauth-client-id

# Create version if missing
gcloud secrets versions add dreamboard-oauth-client-id \
  --data-file=- <<< "YOUR_OAUTH_CLIENT_ID"

# Use latest version in code
version = "latest"
```

---

## Database Issues

### "Firestore connection timeout"

**Symptom:**
```
Error: Failed to connect to Firestore: deadline exceeded
```

**Cause:** Firestore unavailable or network issue

**Solution:**
```bash
# Check Firestore status
gcloud firestore databases list

# Check service account permissions
gcloud projects get-iam-policy radiant-tide-401723 \
  --flatten="bindings[].members" \
  --filter="bindings.role:roles/datastore.user"

# Add role if missing
gcloud projects add-iam-policy-binding radiant-tide-401723 \
  --member="serviceAccount:dreamboard-account@radiant-tide-401723.iam.gserviceaccount.com" \
  --role="roles/datastore.user"
```

---

### "Quota exceeded"

**Symptom:**
```
Error: Cloud Firestore quota exceeded for operation
```

**Cause:** Too many concurrent requests

**Solution:**
1. Wait for quota reset (quota resets daily)
2. Upgrade to higher plan
3. Optimize queries to reduce throughput:
   ```python
   # Instead of: return all docs and filter in code
   query = db.collection('stories').where('status', '==', 'active').limit(100)
   
   # Better: filter in database
   ```

---

## GCS (Cloud Storage) Issues

### "Bucket name case sensitivity"

**Symptom:**
```
Error: 404 Invalid Bucket Name
```

**Cause:** GCS bucket names must be lowercase

**Solution:**
```bash
# Bucket name must be lowercase
# Correct: monks-dreamboard
# Wrong: Monks-Dreamboard

# Check your bucket
gsutil ls gs://monks-dreamboard/

# If wrong bucket created, delete and recreate
gsutil rm -r gs://Monks-Dreamboard/
gsutil mb gs://monks-dreamboard
```

---

## Monitoring Issues

### "Dashboard not showing data"

**Symptom:** Monitoring dashboard shows "No data available"

**Cause:** 
1. Services just deployed (metrics need 5-10 min)
2. No traffic to services
3. Metrics not configured

**Solution:**
```bash
# Wait 10 minutes after first deployment
sleep 600

# Generate some traffic
curl https://dreamboard-backend-XXXX.southamerica-east1.run.app/health

# Check metrics are being collected
gcloud monitoring metrics list --format='value(type)'
```

---

### "Alerts not firing"

**Symptom:** Alert policy configured but no notifications received

**Cause:**
1. Notification channel email not confirmed
2. Alert threshold never reached
3. Notification channel disabled

**Solution:**
```bash
# List notification channels
gcloud alpha monitoring channels list

# Check status - should be VERIFIED
gcloud alpha monitoring channels describe CHANNEL_ID

# Resend verification email
gcloud alpha monitoring channels verify CHANNEL_ID

# Test alert by exceeding threshold
for i in {1..1000}; do curl SERVICE_URL/invalid-endpoint; done
```

---

## Common Error Messages

| Error | Cause | Fix |
|-------|-------|-----|
| `404 Not Found` | Resource doesn't exist | Check resource exists, correct name/ID |
| `403 Permission Denied` | IAM role missing | Add required role to service account |
| `500 Internal Server Error` | Application bug | Check logs, redeploy |
| `503 Service Unavailable` | Service overloaded or down | Scale up, check health, rollback |
| `Deadline exceeded` | Timeout | Increase timeout, optimize code |
| `Resource exhausted` | Quota exceeded | Wait for reset, upgrade plan |

---

## Getting Help

**Resources:**
- [Cloud Run Troubleshooting](https://cloud.google.com/run/docs/troubleshooting)
- [Firestore Troubleshooting](https://cloud.google.com/firestore/docs/troubleshoot)
- [Cloud Logging](https://cloud.google.com/logging/docs/analyze)

**Contact:**
- Gustavo Hervas: gustavo.hervas@monks.com
- Issue: Create GitHub issue with error logs

---

**Last Updated:** 2026-09-28  
**Version:** 1.0
