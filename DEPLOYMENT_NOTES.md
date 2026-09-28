# Production Deployment Notes

This project is deployed manually to Cloud Run using the original
`backend/deploy_backend.sh` and `frontend/deploy_frontend.sh` scripts —
there is no CI/CD pipeline. This file records the real values used for
this project's GCP setup, so a redeploy reproduces the working
configuration instead of guessing at prompts.

## Current environment

| Parameter | Value |
|---|---|
| GCP Project ID | `radiant-tide-401723` |
| Cloud Run region | `southamerica-east1` (São Paulo) |
| Vertex AI location | `us-central1` |
| GCS bucket | `monks-dreamboard` |
| Firestore database | `dreamboard-db` |
| Service account | `dreamboard-sa@radiant-tide-401723.iam.gserviceaccount.com` |
| Backend URL | `https://dreamboard-backend-654852193118.southamerica-east1.run.app` |
| Frontend URL | `https://dreamboard-frontend-654852193118.southamerica-east1.run.app` |
| OAuth Client ID | `654852193118-80b6klec3rsmingglbvie47re4d15sad.apps.googleusercontent.com` |

### Why Cloud Run region and Vertex AI location differ

Vertex AI's GenAI models used here (Veo, newer Imagen versions) are not
available in `southamerica-east1`. The Cloud Run services stay in
`southamerica-east1` for latency/data residency, while the backend's
`LOCATION` env var (used for non-"global" Vertex AI calls) is set to
`us-central1`. `backend/deploy_backend.sh` prompts for these two
separately (`REGION` and `VERTEX_LOCATION`) for exactly this reason —
don't collapse them back into a single value.

### GCS bucket name

Bucket names must be all-lowercase. `monks-dreamboard` is a shared
bucket, not the project-derived default (`${PROJECT_ID}-dreamboard`),
so `deploy_backend.sh` prompts for the bucket name instead of deriving
it automatically.

## Redeploying

### Backend

```bash
cd backend
./deploy_backend.sh
```

When prompted, use: region `southamerica-east1`, Vertex AI location
`us-central1`, bucket `monks-dreamboard`.

### Frontend

```bash
cd frontend
./deploy_frontend.sh \
  radiant-tide-401723 \
  monks-dreamboard \
  dreamboard-sa@radiant-tide-401723.iam.gserviceaccount.com \
  southamerica-east1 \
  https://dreamboard-backend-654852193118.southamerica-east1.run.app \
  654852193118-80b6klec3rsmingglbvie47re4d15sad.apps.googleusercontent.com
```

This regenerates `frontend/dreamboard/src/environments/environment.ts`
and `environment.development.ts` from `environment-template.ts` with
the values above before building.
