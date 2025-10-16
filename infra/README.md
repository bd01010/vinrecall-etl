# VinRecall ETL – Phase 2 Cloud Run/SQL/Scheduler Setup

This repo contains the ETL service that polls NHTSA, writes to Cloud SQL, and opens PRs to the site repo. Deploy matches the Phase 1 design (Cloud Run + Cloud SQL + Cloud Scheduler).

## Prereqs
- GCP project and billing enabled
- Artifact Registry enabled
- Cloud SQL Postgres instance
- Secret Manager for DATABASE_URL, GITHUB_TOKEN
- Service account JSON in GitHub repo secret `GCP_SA_KEY`
- Repo secrets: `GCP_PROJECT_ID`, `GCP_REGION`

## One-time setup (copy/paste)
```bash
PROJECT_ID=<your-project>
REGION=<your-region>   # e.g. us-central1
SERVICE=vinrecall-etl
REPOSITORY=vinrecall
IMAGE=etl

# (Optional) Artifact Registry repo (Docker)
gcloud artifacts repositories create $REPOSITORY \
  --repository-format=docker --location=$REGION --description='VinRecall ETL images'

# Cloud SQL (Postgres)
gcloud sql instances create vinrecall-sql --database-version=POSTGRES_14 --region=$REGION
gcloud sql databases create vinrecall --instance=vinrecall-sql
# Create user and capture url format if using TCP:
# postgres://etluser:<password>@/<db>?host=/cloudsql/$PROJECT_ID:$REGION:vinrecall-sql

# Secret Manager (values to be set)
# DATABASE_URL = connection string for vinrecall DB
# GITHUB_TOKEN = token with contents:write on bd01010/vinrecall

# First build & deploy (same as GitHub workflow)
gcloud builds submit --tag $REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:manual
gcloud run deploy $SERVICE \
  --image $REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:manual \
  --region $REGION --platform managed --allow-unauthenticated=false \
  --set-secrets 'DATABASE_URL=DATABASE_URL:latest,GITHUB_TOKEN=GITHUB_TOKEN:latest' \
  --add-cloudsql-instances $PROJECT_ID:$REGION:vinrecall-sql

# Get service URL
gcloud run services describe $SERVICE --region $REGION --format='value(status.url)'

# Cloud Scheduler (hourly poll)
RUN_URL=$(gcloud run services describe $SERVICE --region $REGION --format='value(status.url)')
gcloud scheduler jobs create http vinrecall-hourly \
  --schedule='0 * * * *' \
  --uri="$RUN_URL/tasks/poll?nhtsa=delta" \
  --http-method=GET \
  --oidc-service-account-email $SERVICE@$PROJECT_ID.iam.gserviceaccount.com
```

## After merge
1) Set repo secrets in GitHub: `GCP_PROJECT_ID`, `GCP_REGION`, `GCP_SA_KEY`.
2) Create Secret Manager secrets: `DATABASE_URL`, `GITHUB_TOKEN`.
3) Run the README commands above once to bootstrap infra.
4) Trigger the GitHub workflow manually (workflow_dispatch) or push a `deploy-*` tag to build & deploy.

## Notes
- Backend changes land via PRs (safe by default). The site repo continues to deploy via Netlify from `main` per the MCP workflow.
