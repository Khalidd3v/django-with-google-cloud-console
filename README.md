# Professional Cloud Run + Cloud SQL (Postgres) + Secrets + Logging + GCS (Django)

This guide rebuilds and redeploys your Django app from scratch using a modern, “standard” setup:

- Docker image built with Cloud Build
- Artifact Registry (Docker repository)
- Cloud Run service (Gunicorn)
- Cloud SQL for PostgreSQL (via Cloud Run Cloud SQL integration)
- Secret Manager (inject secrets as environment variables)
- Cloud Logging (stdout/stderr + view in Console)
- Cloud Storage for Django media via `django-storages`

## 0) What you should expect

- After deployment, opening your Cloud Run URL in the browser should show your home page (currently at `/`) with: `It worked Khalid.`
- API remains under `/api/` (authenticated endpoints).

## 1) Prerequisites (one-time on your machine)

1. Install/enable Google Cloud SDK.
2. Authenticate:
   ```bash
   gcloud auth login
   gcloud auth application-default login
   ```
3. Set your project:
   ```bash
   PROJECT_ID="academic-moon-466815-n6"
   gcloud config set project "$PROJECT_ID"
   ```

## 2) Enable required APIs (one-time per project)

Run:
```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  cloudscheduler.googleapis.com \
  logging.googleapis.com \
  storage-api.googleapis.com
```

Note: `cloudscheduler.googleapis.com` is only needed if you later add schedules; you can remove it if you want. The rest are the core ones.

## 3) Define variables (edit values if you want)

In your terminal:
```bash
REGION="asia-south2"
SERVICE_NAME="django-todo-app"

# Artifact Registry repository (Docker)
AR_REPO="django-todo-app-repo"

# Cloud SQL instance (Postgres)
SQL_INSTANCE_NAME="django-todo-app-postgres"
SQL_DB_NAME="tododb"
SQL_DB_USER="todoappuser"

# GCS bucket for Django media
GCS_BUCKET_NAME="your-unique-gcs-bucket-name"

# Connection name used by Cloud Run / Cloud SQL integration
INSTANCE_CONNECTION_NAME="${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}"
```

## 3.1) Django environment variables (what to set)

Django expects standard Postgres keys:

```bash
DB_ENGINE=postgres
DB_HOST=/cloudsql/${INSTANCE_CONNECTION_NAME}
DB_PORT=5432
DB_NAME=${SQL_DB_NAME}
DB_USER=${SQL_DB_USER}
DB_PASSWORD=... (comes from Secret Manager)

SECRET_KEY=... (comes from Secret Manager)
GS_BUCKET_NAME=${GCS_BUCKET_NAME}
ALLOWED_HOSTS=*
DEBUG=false
```

For local development you can use the provided `.env.example` (SQLite).

## 4) Create Artifact Registry repo (one-time)

```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION"
```

Your image URI will be:

```bash
IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/django-todo-app"
```

## 5) Create Cloud SQL (Postgres) instance + database user (one-time)

### 5.1) Create the instance

```bash
gcloud sql instances create "$SQL_INSTANCE_NAME" \
  --database-version=POSTGRES_16 \
  --tier=db-f1-micro \
  --region="$REGION" \
  --quiet
```

### 5.2) Create a DB password and store it in Secret Manager

Set a password locally (you can paste a strong value):
```bash
DB_PASSWORD="PUT_A_STRONG_PASSWORD_HERE"
```

Create the secret:
```bash
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=<(printf "%s" "$DB_PASSWORD")
```
If your shell does not support `<( ... )`, replace with:
```bash
printf "%s" "$DB_PASSWORD" > /tmp/db_password.txt
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 5.3) Create database + user inside Cloud SQL

Create the DB:
```bash
gcloud sql databases create "$SQL_DB_NAME" --instance="$SQL_INSTANCE_NAME"
```

Create the user:
```bash
gcloud sql users create "$SQL_DB_USER" \
  --instance="$SQL_INSTANCE_NAME" \
  --password="$DB_PASSWORD"
```

## 6) Create secrets for the app (one-time)

### 6.1) SECRET_KEY secret

Set it locally:
```bash
SECRET_KEY_VALUE="PUT_A_SECURE_SECRET_KEY_HERE"
```

Create secret:
```bash
printf "%s" "$SECRET_KEY_VALUE" > /tmp/secret_key.txt
gcloud secrets create "django-todo-secret-key" \
  --replication-policy="automatic" \
  --data-file=/tmp/secret_key.txt
rm /tmp/secret_key.txt
```

### 6.2) DB_PASSWORD secret

We will store only the database password in Secret Manager.

Your Django production settings use standard Django DB env vars:
`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, and `DB_PASSWORD`.

```bash
DB_PASSWORD_SECRET_NAME="django-todo-db-password"
```

Create the secret:
```bash
printf "%s" "$DB_PASSWORD" > /tmp/db_password.txt
gcloud secrets create "$DB_PASSWORD_SECRET_NAME" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 6.3) Django superuser password secret (optional but recommended)

This app has a management command `create_superuser_if_missing` to create an admin user automatically.

Set values locally:
```bash
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"
DJANGO_SUPERUSER_PASSWORD_VALUE="PUT_A_STRONG_PASSWORD_HERE"
```

Create the superuser password secret:
```bash
printf "%s" "$DJANGO_SUPERUSER_PASSWORD_VALUE" > /tmp/superuser_password.txt
gcloud secrets create "django-todo-superuser-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/superuser_password.txt
rm /tmp/superuser_password.txt
```

## 7) Create GCS bucket + IAM (one-time)

```bash
gsutil mb -p "$PROJECT_ID" -l "$REGION" "gs://${GCS_BUCKET_NAME}"
```

Create a dedicated service account for Cloud Run:
```bash
RUN_SA_NAME="django-todo-run-sa"
RUN_SA_EMAIL="${RUN_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$RUN_SA_NAME" \
  --display-name="$RUN_SA_NAME"
```

Grant least-privilege-ish roles:

1. Secret access:
   ```bash
   gcloud secrets add-iam-policy-binding "django-todo-secret-key" \
     --member="serviceAccount:${RUN_SA_EMAIL}" \
     --role="roles/secretmanager.secretAccessor"

   gcloud secrets add-iam-policy-binding "django-todo-db-password" \
     --member="serviceAccount:${RUN_SA_EMAIL}" \
     --role="roles/secretmanager.secretAccessor"

   gcloud secrets add-iam-policy-binding "django-todo-superuser-password" \
     --member="serviceAccount:${RUN_SA_EMAIL}" \
     --role="roles/secretmanager.secretAccessor"
   ```
2. Cloud SQL access:
   ```bash
   gcloud projects add-iam-policy-binding "$PROJECT_ID" \
     --member="serviceAccount:${RUN_SA_EMAIL}" \
     --role="roles/cloudsql.client"
   ```
3. Cloud Storage access (for Django media):
   ```bash
   gsutil iam ch "serviceAccount:${RUN_SA_EMAIL}:objectAdmin" "gs://${GCS_BUCKET_NAME}"
   ```

## 8) Deploy the Docker image to Cloud Run

## 8.1) Build & push the Docker image

From your project folder:

```bash
cd "/Users/khalid/Documents/projects/d3v/Google Cloud Practice/Todo Django App"
```

Build and push:
```bash
IMAGE_TAG="latest"
gcloud builds submit \
  --tag "${IMAGE_URI}:${IMAGE_TAG}" \
  .
```

## 8.2) Deploy Cloud Run service

Set env vars that are not secret and inject secrets at deploy time:
```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$RUN_SA_EMAIL" \
  --add-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=/cloudsql/${INSTANCE_CONNECTION_NAME},DB_PORT=5432,DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest"
```

## 9) Run database migrations (standard approach)

Use a Cloud Run Job so migrations happen in the same runtime environment.

Deploy a one-time job and execute it now:

```bash
JOB_NAME="django-todo-migrate"

gcloud run jobs deploy "$JOB_NAME" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --service-account "$RUN_SA_EMAIL" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=/cloudsql/${INSTANCE_CONNECTION_NAME},DB_PORT=5432,DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest" \
  --command python \
  --args manage.py,migrate \
  --execute-now
```

If the job runs successfully, you should see migration output in job logs.

## 9.1) Create Django superuser (one-time)

Run the superuser creation job (it will skip creation if the user already exists):

```bash
JOB_NAME="django-todo-superuser"

DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"

gcloud run jobs deploy "$JOB_NAME" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --service-account "$RUN_SA_EMAIL" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=/cloudsql/${INSTANCE_CONNECTION_NAME},DB_PORT=5432,DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*,DJANGO_SUPERUSER_USERNAME=${DJANGO_SUPERUSER_USERNAME},DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL}" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest,DJANGO_SUPERUSER_PASSWORD=django-todo-superuser-password:latest" \
  --command python \
  --args manage.py,create_superuser_if_missing \
  --execute-now
```

## 10) Configure/verify Cloud Logging

Cloud Run automatically sends `stdout` and `stderr` to Cloud Logging.

1. View service logs:
   ```bash
   gcloud run services logs read "$SERVICE_NAME" --region "$REGION" --limit 50
   ```
2. In the GCP Console:
   - Logging
   - Log Explorer
   - Filter: `resource.type="cloud_run_revision"` and `service_name="$SERVICE_NAME"`

Tip: refresh the browser and then immediately check logs when debugging startup issues.

## 11) Verify the app is reachable

1. Get the service URL:
   ```bash
   gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)'
   ```
2. Test home page:
   ```bash
   curl -i "https://$SERVICE_URL/"
   ```
3. You should see:
   - HTTP 200
   - HTML containing `It worked Khalid.`

## 12) Notes about Django + Storage

- Your Django code uses `django-storages` when `DEBUG=false` and `GS_BUCKET_NAME` is set.
- Media uploads should go to the GCS bucket.
- Static files are built into the container at image build time and served via WhiteNoise.

## 14) Troubleshooting (common IAM/permission edge cases)

1. `503` / container not staying up:
   - Check service logs:
     ```bash
     gcloud run services logs read "$SERVICE_NAME" --region "$REGION" --limit 50
     ```
   - Look for missing env vars like `DB_PASSWORD`, `SECRET_KEY`, or DB host errors.

2. Secret injection fails (`secret not found` or `permission denied`):
   - Ensure the Cloud Run service account has `roles/secretmanager.secretAccessor` for:
     - `django-todo-secret-key`
     - `django-todo-db-password`
     - `django-todo-superuser-password` (only needed for the superuser job)

3. Cloud SQL connection fails:
   - Ensure the Cloud Run **service account used by both service and jobs** has:
     - `roles/cloudsql.client`
   - Ensure `DB_HOST` is correct:
     - For Cloud Run socket integration:
       `DB_HOST=/cloudsql/${INSTANCE_CONNECTION_NAME}`

4. Migrations/job fails but service logs look fine:
   - The job uses the service account you pass via `--service-account`.
   - Make sure the same account has the same IAM roles and secrets.

5. GCS upload fails (403/permission errors):
   - Ensure your service account has access to the bucket, for example:
     ```bash
     gsutil iam ch "serviceAccount:${RUN_SA_EMAIL}:objectAdmin" "gs://${GCS_BUCKET_NAME}"
     ```

## 13) Cleanup (optional)

If you want to delete everything later, delete in this order:

1. Cloud Run service and jobs
2. Cloud SQL instance
3. Artifact Registry repo
4. GCS bucket (after you confirm you do not need files)
5. Secrets

