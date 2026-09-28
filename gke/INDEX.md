# Dreamboard Deployment Documentation Index

Quick reference for all deployment-related documentation.

## 🚀 Getting Started

**New to this project?** Start here:

1. **[DEPLOYMENT.md](DEPLOYMENT.md)** - Main deployment guide
   - Quick start (5 minutes)
   - Full workflow overview
   - Common operations
   - **Best for:** First-time setup, understanding the overall flow

## 📋 Setup & Implementation

### Phase 1: Infrastructure (Terraform)
- **[PHASE-1-SETUP.md](PHASE-1-SETUP.md)** - Terraform Cloud Run
  - Prerequisites
  - Step-by-step setup
  - Verification
  - **Time:** 1-2 hours
  - **Status:** ✅ Complete

### Phase 2: CI/CD (GitHub Actions)
- **[PHASE-2-SETUP.md](PHASE-2-SETUP.md)** - Automated deployments
  - Workload Identity Federation setup
  - GitHub secrets configuration
  - Workflow testing
  - **Time:** 1-2 hours
  - **Status:** ✅ Complete

### Phase 3: Security & Monitoring
- **[PHASE-3-SETUP.md](PHASE-3-SETUP.md)** - Secrets & Monitoring
  - Secret Manager configuration
  - Cloud Monitoring dashboard
  - Alert policies
  - **Time:** 1-2 hours
  - **Status:** ✅ Complete

## 🔧 Operations & Maintenance

### Daily Operations
- **[MAINTENANCE.md](MAINTENANCE.md)** - Routine tasks
  - Daily health checks
  - Weekly reviews
  - Monthly security audits
  - Incident response
  - **Best for:** Ops team, on-call engineers

### Troubleshooting
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** - Common issues
  - Deployment errors
  - Cloud Run issues
  - GitHub Actions failures
  - Database problems
  - **Best for:** When something breaks

### Integration Guides
- **[terraform/SECRETS-INTEGRATION.md](terraform/SECRETS-INTEGRATION.md)** - Using Secret Manager
  - Before/after comparison
  - Integration steps
  - Verification
  - **Best for:** Understanding secrets architecture

## 📚 Reference Materials

### Terraform Files
```
terraform/
├── main.tf              # Main configuration
├── cloud-run.tf         # Cloud Run services
├── secrets.tf           # Secret Manager
├── monitoring.tf        # Cloud Monitoring
├── iam.tf              # IAM roles
├── variables.tf        # Input variables
├── outputs.tf          # Outputs
├── backend.tf          # State backend
└── versions.tf         # Provider versions
```

### GitHub Workflows
```
.github/workflows/
├── build-images.yml    # Build Docker images
├── deploy.yml          # Deploy with Terraform
├── validate-pr.yml     # Validate PRs
└── README.md           # Workflow documentation
```

## 🎯 By Role

### DevOps Engineer
- [DEPLOYMENT.md](DEPLOYMENT.md) - Overall architecture
- [PHASE-1-SETUP.md](PHASE-1-SETUP.md) - Infrastructure setup
- [MAINTENANCE.md](MAINTENANCE.md) - Daily operations
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Problem resolution

### Software Engineer
- [DEPLOYMENT.md](DEPLOYMENT.md) - Quick start
- [PHASE-2-SETUP.md](PHASE-2-SETUP.md) - CI/CD workflows
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues

### Security Engineer
- [PHASE-3-SETUP.md](PHASE-3-SETUP.md) - Secret management
- [terraform/SECRETS-INTEGRATION.md](terraform/SECRETS-INTEGRATION.md) - Secret architecture
- [MAINTENANCE.md](MAINTENANCE.md) - Monthly security audit

### Site Reliability Engineer (SRE)
- [DEPLOYMENT.md](DEPLOYMENT.md) - Architecture overview
- [MAINTENANCE.md](MAINTENANCE.md) - Operational runbooks
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Incident response
- [terraform/monitoring.tf](terraform/monitoring.tf) - Alert policies

## ❓ FAQ

### Q: How do I deploy new code?
**A:** Push to `main` branch. GitHub Actions automatically builds, tests, and deploys via Terraform.

### Q: How do I add an environment variable?
**A:** Edit `gke/terraform/cloud-run.tf`, add to `env` section, then run `terraform apply -auto-approve`.

### Q: How do I rotate secrets?
**A:** See [PHASE-3-SETUP.md](PHASE-3-SETUP.md) → Secret Rotation section.

### Q: Service is down. What do I do?
**A:** See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) → "When Service is Down" section.

### Q: How do I check if everything is working?
**A:** See [MAINTENANCE.md](MAINTENANCE.md) → "Daily Operations" → "Health Check" section.

### Q: How do I rollback a deployment?
**A:** See [DEPLOYMENT.md](DEPLOYMENT.md) → "Rollback to Previous Version" section.

### Q: Can I deploy without GitHub Actions?
**A:** Yes, manually run `terraform apply -auto-approve` in `gke/terraform/`.

### Q: What if Terraform state gets corrupted?
**A:** See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) → "State bucket not found" section.

## 📊 Deployment Status

| Phase | Topic | Status | Time |
|-------|-------|--------|------|
| **1** | Terraform Cloud Run | ✅ Complete | 1-2h |
| **2** | GitHub Actions CI/CD | ✅ Complete | 1-2h |
| **3** | Secrets & Monitoring | ✅ Complete | 1-2h |
| **4** | Documentation | ✅ Complete | - |

---

## 🔗 Related Documentation

- [Dreamboard Backend README](../../backend/README.md)
- [Dreamboard Frontend README](../../frontend/README.md)
- [Main README](../../README.md)

## 📞 Support

**Issues?** Check:
1. [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for known issues
2. GitHub Issues for reported problems
3. Contact: gustavo.hervas@monks.com

## 📝 Document Versioning

| Date | Version | Changes |
|------|---------|---------|
| 2026-09-28 | 1.0 | Initial: Phases 1-3 (IaC, CI/CD, Secrets/Monitoring) |

---

**Last Updated:** 2026-09-28  
**Maintainer:** Gustavo Hervas  
**Status:** 🟢 Production Ready
