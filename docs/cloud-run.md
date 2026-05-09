# Google Cloud Run Deployment

ZoAuth runs natively on Cloud Run — stateless, auto-scaling, pay-per-request.

---

## Prerequisites

- `gcloud` CLI configured (`gcloud auth login`)
- Cloud SQL PostgreSQL instance (or external Postgres)
- Secret Manager enabled on your project

---

## 1. Store secrets in Secret Manager

```bash
PROJECT=your-gcp-project
REGION=us-central1

# Create secrets
echo -n "$(openssl rand -base64 32)" | \
  gcloud secrets create zoauth-jwk-password --data-file=- --project=$PROJECT

echo -n "$(openssl rand -hex 8)" | \
  gcloud secrets create zoauth-jwk-salt --data-file=- --project=$PROJECT

echo -n "postgres://zoauth:yourpassword@/zoauth?host=/cloudsql/$PROJECT:$REGION:your-instance&sslmode=disable" | \
  gcloud secrets create zoauth-database-url --data-file=- --project=$PROJECT

echo -n "change-me-now" | \
  gcloud secrets create zoauth-admin-password --data-file=- --project=$PROJECT
```

---

## 2. Deploy to Cloud Run

```bash
gcloud run deploy zoauth \
  --image ghcr.io/zoauth/go-auth-engine:latest \
  --region $REGION \
  --project $PROJECT \
  --platform managed \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 10 \
  --memory 512Mi \
  --cpu 1 \
  --port 8080 \
  --add-cloudsql-instances $PROJECT:$REGION:your-instance \
  --set-env-vars "JWT_ISSUER=https://auth.yourdomain.com" \
  --set-env-vars "SPRING_ENABLED=false" \
  --set-env-vars "AUTHCODE_NATIVE=true,LOGOUT_NATIVE=true,CONSENT_NATIVE=true" \
  --set-env-vars "METRICS_ENABLED=true" \
  --set-env-vars "LOG_FORMAT=json" \
  --set-env-vars "SEED_DEFAULT_TENANT=false" \
  --set-secrets "JWK_ENCRYPTION_PASSWORD=zoauth-jwk-password:latest" \
  --set-secrets "JWK_ENCRYPTION_SALT=zoauth-jwk-salt:latest" \
  --set-secrets "DATABASE_URL=zoauth-database-url:latest" \
  --service-account your-sa@$PROJECT.iam.gserviceaccount.com
```

---

## 3. Map your custom domain

```bash
gcloud run domain-mappings create \
  --service zoauth \
  --domain auth.yourdomain.com \
  --region $REGION \
  --project $PROJECT
```

---

## Service Account Permissions

The Cloud Run service account needs:
```bash
SA=your-sa@$PROJECT.iam.gserviceaccount.com

# Read secrets
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:$SA" --role="roles/secretmanager.secretAccessor"

# Connect to Cloud SQL
gcloud projects add-iam-policy-binding $PROJECT \
  --member="serviceAccount:$SA" --role="roles/cloudsql.client"
```

---

## Health Check

Cloud Run uses the `/health/live` endpoint automatically (HTTP probe on port 8080).

For startup probe (gives ZoAuth time to run migrations on cold start):
```bash
gcloud run services update zoauth \
  --startup-probe httpGet.path=/health/live,initialDelaySeconds=10,periodSeconds=5 \
  --region $REGION --project $PROJECT
```

---

## Multi-region / High Availability

Deploy to multiple regions with a Global Load Balancer:

```bash
# Deploy to additional region
gcloud run deploy zoauth \
  --image ghcr.io/zoauth/go-auth-engine:latest \
  --region europe-west1 \
  --project $PROJECT \
  [same flags as above]
```

Ensure all instances share the **same** `JWK_ENCRYPTION_PASSWORD` and point to the **same** PostgreSQL database (use Cloud SQL with read replicas or AlloyDB).

---

## Continuous Deployment

ZoAuth's existing Cloud Build pipeline (`cloudbuild.yaml`) handles CI/CD automatically on every push to `master`.
