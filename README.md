# Django on Cloud Run (Docker) with Cloud SQL + Secrets + GCS

CLI-first, first-time guide for deploying this Django app.

This guide assumes your Django settings use standard DB env vars:
- `DB_ENGINE`
- `DB_HOST`
- `DB_PORT`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

And your app uses:
- Secret Manager injection for `SECRET_KEY`, `DB_PASSWORD`, and `CSRF_TRUSTED_ORIGINS` (secret `django-todo-csrf-trusted-origins`)
- Cloud SQL socket integration (`DB_HOST=/cloudsql/...`)
- Cloud Storage for media (`GS_BUCKET_NAME`)

## 0) One-time setup (variables)
Run this exactly (do not rename variable names used below):
```bash
PROJECT_ID="academic-moon-466815-n6"
gcloud config set project "$PROJECT_ID"

REGION="asia-south2"

SERVICE_NAME="django-todo-app"

# Cloud SQL
SQL_INSTANCE_NAME="django-todo-app-postgres"
SQL_DB_NAME="tododb"
SQL_DB_USER="todoappuser"

# Artifact Registry (Docker repo)
AR_REPO="django-todo-app-repo"
IMAGE_NAME="django-todo-app"
IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}"
IMAGE_TAG="latest"

# GCS bucket (must be bucket name only, not gs://...)
GCS_BUCKET_NAME="django-todo-app"

INSTANCE_CONNECTION_NAME="${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}"
DB_HOST="/cloudsql/${INSTANCE_CONNECTION_NAME}"
DB_PORT="5432"

# Service account email (for Cloud Run + Jobs)
RUN_SA_NAME="django-todo-run-sa"
RUN_SA_EMAIL="${RUN_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

# Superuser
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"

# CSRF: use Secret Manager secret django-todo-csrf-trusted-origins (see §3.4), not a shell var here
```

## 1) Enable required APIs
```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  logging.googleapis.com \
  storage-api.googleapis.com
```

## 2) Create Cloud SQL (Postgres)
```bash
gcloud sql instances create "$SQL_INSTANCE_NAME" \
  --database-version=POSTGRES_16 \
  --tier=db-perf-optimized-N-2 \
  --region="$REGION" \
  --quiet

gcloud sql databases create "$SQL_DB_NAME" --instance="$SQL_INSTANCE_NAME"
```

Create DB user + password:
```bash
DB_PASSWORD_VALUE='PUT_A_STRONG_PASSWORD_HERE'

gcloud sql users create "$SQL_DB_USER" \
  --instance="$SQL_INSTANCE_NAME" \
  --password="$DB_PASSWORD_VALUE"
```

## 3) Create secrets in Secret Manager
### 3.1 `SECRET_KEY`
```bash
SECRET_KEY_VALUE='PUT_A_SECURE_SECRET_KEY_HERE'
printf "%s" "$SECRET_KEY_VALUE" > /tmp/secret_key.txt
gcloud secrets create "django-todo-secret-key" \
  --replication-policy="automatic" \
  --data-file=/tmp/secret_key.txt
rm /tmp/secret_key.txt
```

### 3.2 `DB_PASSWORD`
```bash
printf "%s" "$DB_PASSWORD_VALUE" > /tmp/db_password.txt
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 3.3 Superuser password (recommended)
```bash
DJANGO_SUPERUSER_PASSWORD_VALUE='PUT_A_SUPERUSER_PASSWORD_HERE'

printf "%s" "$DJANGO_SUPERUSER_PASSWORD_VALUE" > /tmp/superuser_password.txt
gcloud secrets create "django-todo-superuser-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/superuser_password.txt
rm /tmp/superuser_password.txt
```

### 3.4 `CSRF_TRUSTED_ORIGINS` (admin login — store in Secret Manager)

Cloud Run injects this as the **`CSRF_TRUSTED_ORIGINS`** env var. The secret value must be your **exact** service URL (no path, no trailing slash), e.g. `https://django-todo-app-XXXX.asia-south2.run.app`.

**After the service exists**, set the secret. Cloud Run often exposes **two** URLs (`*.asia-south2.run.app` and `*.a.run.app`); list **both** comma-separated so admin works from either:

```bash
# All URLs from the service (recommended)
URLS_JSON="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(metadata.annotations.run.googleapis.com/urls)')"
python3 -c "import json,sys; u=json.loads(sys.argv[1]); print(','.join(x.rstrip('/') for x in u))" "$URLS_JSON" | gcloud secrets create "django-todo-csrf-trusted-origins" \
  --replication-policy=automatic --data-file=-
```

Or a single URL:

```bash
CSRF_ORIGIN="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)')"
printf '%s' "${CSRF_ORIGIN%/}" | gcloud secrets create "django-todo-csrf-trusted-origins" \
  --replication-policy=automatic --data-file=-
```

If the secret already exists (update after URL change or second hostname):

```bash
printf '%s' "https://YOUR-EXACT-ORIGIN" | gcloud secrets versions add "django-todo-csrf-trusted-origins" --data-file=-
```

**Two origins** (e.g. `*.run.app` and `*.a.run.app`): put both in one secret, comma-separated, no spaces:

```bash
printf '%s' "https://host1.asia-south2.run.app,https://host2.a.run.app" | gcloud secrets versions add "django-todo-csrf-trusted-origins" --data-file=-
```

Grant the Cloud Run service account access (same as other secrets):

```bash
gcloud secrets add-iam-policy-binding "django-todo-csrf-trusted-origins" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"
```

**Redeploy script** (build + push + deploy with CSRF from this secret): `scripts/redeploy-cloud-run-csrf-secret.sh`  
(If the service does not exist yet, set `CSRF_TRUSTED_ORIGINS_OVERRIDE=https://...` once before the first deploy.)

## 4) Create GCS bucket (media)
```bash
gsutil mb -p "$PROJECT_ID" -l "$REGION" "gs://${GCS_BUCKET_NAME}"
```

## 5) Create Artifact Registry repo
```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION" \
  --quiet
```

## 6) Build + push Docker image
```bash
cd "/Users/khalid/Documents/projects/d3v/Google Cloud Practice/Todo Django App"

gcloud builds submit \
  --tag "${IMAGE_URI}:${IMAGE_TAG}" \
  .
```

## 6.1 Dockerfile + settings.py (working way)
This app uses `whitenoise.storage.CompressedManifestStaticFilesStorage` when `DEBUG=false` (see `settings.py`), which requires `staticfiles/manifest.json`.

So your **image build must run** `collectstatic` with `DEBUG=False` to generate that manifest. If you build with `DEBUG=True` and then run with `DEBUG=false`, admin assets can break and `/admin/*` may return **500**.

### Working `Dockerfile`
```dockerfile
# --- Builder Stage ---
FROM python:3.12-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libpq-dev && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# --- Runner Stage ---
FROM python:3.12-slim
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PORT=8080

# Ensure Django project package is always on Python path (fixes Cloud Run import errors)
ENV PYTHONPATH=/app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache-dir /wheels/*
COPY . .

# Run collectstatic with DEBUG=false so the production static storage backend
# (CompressedManifestStaticFilesStorage) generates `staticfiles/manifest.json`.
# We still use SQLite during build because Cloud Run runtime DB secrets/vars
# are not available at image-build time.
RUN SECRET_KEY=build-only-key DEBUG=False DB_ENGINE=sqlite DB_NAME=db.sqlite3 python manage.py collectstatic --noinput

RUN addgroup --system django && adduser --system --group django && \
    chown -R django:django /app
USER django

# Start Gunicorn directly (no extra entrypoint script required).
CMD ["/bin/sh", "-c", "export PYTHONPATH=/app; cd /app; exec gunicorn --chdir /app todo_project.wsgi:application --bind 0.0.0.0:${PORT:-8080} --workers 2 --threads 4 --timeout 0"]
```

### Working `todo_project/settings.py` parts
DB config (Cloud SQL / Postgres) uses standard env vars:
```python
env = environ.Env(
    # set casting, default value
    DEBUG=(bool, False)
)

# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = env('SECRET_KEY')

# SECURITY WARNING: don't run with debug turned on in production!
DEBUG = env('DEBUG')

ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=['*'])

# Comma-separated in env/secrets, full URL with scheme, e.g. https://svc-hash.region.run.app
CSRF_TRUSTED_ORIGINS = env.list('CSRF_TRUSTED_ORIGINS', default=[])

DB_ENGINE = env('DB_ENGINE', default='sqlite' if DEBUG else 'postgres')

if DB_ENGINE == 'sqlite':
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
            'NAME': env('DB_NAME', default=str(BASE_DIR / 'db.sqlite3')),
        }
    }
else:
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': env('DB_NAME'),
            'USER': env('DB_USER'),
            'PASSWORD': env('DB_PASSWORD'),
            'HOST': env('DB_HOST'),
            'PORT': env('DB_PORT', default='5432'),
        }
    }
```

Static/media storage switches based on `DEBUG`:
```python
if DEBUG:
    STORAGES = {
        "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
        "staticfiles": {"BACKEND": "django.contrib.staticfiles.storage.StaticFilesStorage"},
    }
else:
    GS_BUCKET_NAME = env('GS_BUCKET_NAME', default=None)

    if GS_BUCKET_NAME:
        STORAGES = {
            "default": {"BACKEND": "storages.backends.gcloud.GoogleCloudStorage"},
            "staticfiles": {
                "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
            },
        }
        GS_DEFAULT_ACL = 'publicRead'
    else:
        STORAGES = {
            "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
            "staticfiles": {
                "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
            },
        }
```

Important: when redeploying Cloud Run, make sure you set **all** required env vars together (`DB_NAME`, `DB_USER`, `DB_ENGINE`, `DB_HOST`, etc.). Using `gcloud run deploy --set-env-vars` with only `DEBUG=false` can accidentally remove/overwrite the other env vars in the running service.

If you change `settings.py`, run **Step 6** again (rebuild image) before redeploying, or Cloud Run will still run the old code.

## 6.2 CSRF trusted origins (fix **403** on `/admin` login)

Browsers send an **`Origin`** header on `POST`. Django must see that origin in **`CSRF_TRUSTED_ORIGINS`**. If not:

> **Forbidden (403) – CSRF verification failed** / **Origin checking failed – … does not match any trusted origins**

**Recommended (this project):** Store the value in Secret Manager as **`django-todo-csrf-trusted-origins`** and map it at deploy time to env var **`CSRF_TRUSTED_ORIGINS`** (see **§3.4** and **`scripts/redeploy-cloud-run-csrf-secret.sh`**). That avoids comma issues in `gcloud --set-env-vars` and keeps the origin out of plain env.

**Why Cloud Run:** `ALLOWED_HOSTS=*` does **not** trust every origin for CSRF. If you use two URLs (`*.region.run.app` and `*.a.run.app`), put **both** in the secret, comma-separated, no spaces.

### Secret value format (exactly what Django needs)

- **Scheme + host** — e.g. `https://django-todo-app-648441471321.asia-south2.run.app`
- **No** path, **no** trailing slash
- **Two URLs:** `https://first.run.app,https://second.a.run.app`

Local dev: set **`CSRF_TRUSTED_ORIGINS`** in `.env` (same format). Production Cloud Run: use the secret + `--set-secrets ... CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:latest`.

### Discover your URL

```bash
gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)'
```

### Confirm the rejected origin

Set **`DEBUG=true`** briefly, submit the admin form again, and read the **Reason given for failure** — it shows the exact `Origin` to add.

### After CSRF: **500** on submit

Usually **database** (wrong `DB_PASSWORD` vs Cloud SQL user, or missing DB). Use `DEBUG=true` or logs; align secret and `gcloud sql users set-password`, ensure database exists, run migrations.

---

## 7) IAM (service account roles)
This service account must be able to:
- read secrets
- connect to Cloud SQL
- write to your GCS bucket

```bash
gcloud iam service-accounts create "$RUN_SA_NAME" \
  --display-name="$RUN_SA_NAME" || true

gcloud secrets add-iam-policy-binding "django-todo-secret-key" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-db-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-superuser-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-csrf-trusted-origins" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/cloudsql.client"

gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

## 8) Deploy Cloud Run service (CLI)
`CSRF_TRUSTED_ORIGINS` is injected from Secret Manager (secret `django-todo-csrf-trusted-origins` — see §3.4).

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$RUN_SA_EMAIL" \
  --add-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}"
```

## 9) Run migrations (Cloud Run Job)
```bash
gcloud run jobs deploy "django-todo-migrate" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}" \
  --command python \
  --args manage.py,migrate \
  --execute-now
```

## 10) Create Django superuser (Cloud Run Job)
```bash
gcloud run jobs deploy "django-todo-superuser" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*,DJANGO_SUPERUSER_USERNAME=${DJANGO_SUPERUSER_USERNAME},DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL}" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},DJANGO_SUPERUSER_PASSWORD=django-todo-superuser-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}" \
  --command python \
  --args manage.py,create_superuser_if_missing \
  --execute-now
```

**Admin login issues:** CSRF **403** and DB **500** — see **§ 6.2 CSRF trusted origins** above.

## 11) Verify + Cloud Logging
```bash
gcloud run services logs read "$SERVICE_NAME" --region "$REGION" --limit 50

SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)')"
curl -i "https://${SERVICE_URL##*/}/"
```

If everything is correct, `/` should return `It worked Khalid.`.

# Django on Cloud Run (Docker) + Cloud SQL (Postgres) + Secrets + GCS

Clean **CLI-first** guide (intended for first-time users). It assumes your Django settings use standard DB env vars:
- `DB_ENGINE`
- `DB_HOST`
- `DB_PORT`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

And Cloud Run secrets are injected as environment variables (not mounted inside `/app`).

## 0) One-time setup: your shell variables

Run this and do **not** rename the variables (Step 8+ uses the same names):

```bash
PROJECT_ID="academic-moon-466815-n6"
gcloud config set project "$PROJECT_ID"

REGION="asia-south2"

SERVICE_NAME="django-todo-app"

# Cloud SQL
SQL_INSTANCE_NAME="django-todo-app-postgres"
SQL_DB_NAME="tododb"
SQL_DB_USER="todoappuser"

# Artifact Registry (Docker repo)
AR_REPO="django-todo-app-repo"
IMAGE_NAME="django-todo-app"
IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}"

# GCS bucket for Django media
# IMPORTANT: bucket name ONLY, no `gs://`
GCS_BUCKET_NAME="django-todo-app"

# Cloud SQL socket connection name + DB host/port
INSTANCE_CONNECTION_NAME="${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}"
DB_HOST="/cloudsql/${INSTANCE_CONNECTION_NAME}"
DB_PORT="5432"

# Docker tag we will build/push
IMAGE_TAG="latest"

# CSRF: Secret Manager secret django-todo-csrf-trusted-origins → env CSRF_TRUSTED_ORIGINS (see §3.4 in first guide or below)
```

## 1) Enable required APIs

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  logging.googleapis.com \
  storage-api.googleapis.com
```

## 2) Create Cloud SQL (Postgres)

Your project may be **ENTERPRISE_PLUS**, where `db-f1-micro` can fail. Use a supported tier:

```bash
gcloud sql instances create "$SQL_INSTANCE_NAME" \
  --database-version=POSTGRES_16 \
  --tier=db-perf-optimized-N-2 \
  --region="$REGION" \
  --quiet
```

Create DB:
```bash
gcloud sql databases create "$SQL_DB_NAME" --instance="$SQL_INSTANCE_NAME"
```

Create user + store password:
```bash
DB_PASSWORD_VALUE='PUT_A_STRONG_PASSWORD_HERE'   # use single quotes if password has `!`

gcloud sql users create "$SQL_DB_USER" \
  --instance="$SQL_INSTANCE_NAME" \
  --password="$DB_PASSWORD_VALUE"
```

## 3) Create secrets (Secret Manager)

### 3.1) SECRET_KEY
```bash
SECRET_KEY_VALUE='PUT_A_SECURE_SECRET_KEY_HERE'

printf "%s" "$SECRET_KEY_VALUE" > /tmp/secret_key.txt
gcloud secrets create "django-todo-secret-key" \
  --replication-policy="automatic" \
  --data-file=/tmp/secret_key.txt
rm /tmp/secret_key.txt
```

### 3.2) DB_PASSWORD
```bash
printf "%s" "$DB_PASSWORD_VALUE" > /tmp/db_password.txt
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 3.3) Superuser password (recommended)
```bash
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"
DJANGO_SUPERUSER_PASSWORD_VALUE='PUT_A_SUPERUSER_PASSWORD_HERE'

printf "%s" "$DJANGO_SUPERUSER_PASSWORD_VALUE" > /tmp/superuser_password.txt
gcloud secrets create "django-todo-superuser-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/superuser_password.txt
rm /tmp/superuser_password.txt
```

### 3.4) CSRF origins secret (`django-todo-csrf-trusted-origins`)

Same as **§3.4** in the first guide: store your Cloud Run URL (e.g. `https://django-todo-app-648441471321.asia-south2.run.app`) in secret `django-todo-csrf-trusted-origins` and grant `RUN_SA_EMAIL` `secretAccessor` on it.

## 4) Create GCS bucket

```bash
gsutil mb -p "$PROJECT_ID" -l "$REGION" "gs://${GCS_BUCKET_NAME}"
```

## 5) Create Artifact Registry repository (Docker repo)

```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION" \
  --quiet
```

## 6) Build + push Docker image

```bash
cd "/Users/khalid/Documents/projects/d3v/Google Cloud Practice/Todo Django App"

gcloud builds submit \
  --tag "${IMAGE_URI}:${IMAGE_TAG}" \
  .
```

## 7) (Optional but recommended) Service account + IAM

Cloud Run runs as a service account. It needs permissions to:
- read secrets (Secret Manager)
- connect to Cloud SQL
- write media to GCS

If you skip this section, Cloud Run will use the project’s default service account, and your deploy/jobs may fail with permission errors until you grant roles to that default account.

### 7.1) Create service account and grant roles

```bash
RUN_SA_NAME="django-todo-run-sa"
RUN_SA_EMAIL="${RUN_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$RUN_SA_NAME" \
  --display-name="$RUN_SA_NAME"

gcloud secrets add-iam-policy-binding "django-todo-secret-key" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-db-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-superuser-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-csrf-trusted-origins" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/cloudsql.client"

gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

## 8) Deploy Cloud Run service (CLI)

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$RUN_SA_EMAIL" \
  --add-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}"
```

## 9) Run migrations (Cloud Run Job)

```bash
gcloud run jobs deploy "django-todo-migrate" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}" \
  --command python \
  --args manage.py,migrate \
  --execute-now
```

## 10) Create Django superuser (Cloud Run Job)

```bash
gcloud run jobs deploy "django-todo-superuser" \
  --image "${IMAGE_URI}:${IMAGE_TAG}" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*,DJANGO_SUPERUSER_USERNAME=${DJANGO_SUPERUSER_USERNAME},DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL}" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:${IMAGE_TAG},DB_PASSWORD=django-todo-db-password:${IMAGE_TAG},DJANGO_SUPERUSER_PASSWORD=django-todo-superuser-password:${IMAGE_TAG},CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:${IMAGE_TAG}" \
  --command python \
  --args manage.py,create_superuser_if_missing \
  --execute-now
```

## 11) Cloud Logging (debugging)

```bash
gcloud run services logs read "$SERVICE_NAME" --region "$REGION" --limit 50
```

Common causes of `503`:
- DB connection settings wrong
- missing secrets/permission errors
- migrations not run yet

## 12) Verify the app

Service URL:
```bash
gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)'
```

Test home page:
```bash
curl -i "https://$SERVICE_URL/"
```

You should see `It worked Khalid.` in HTML.

# Django on Cloud Run (Docker) + Cloud SQL (Postgres) + Secrets + GCS

Clean, first-time, **CLI-first** guide to deploy this Django app to Google Cloud:
- Cloud Run (Docker, Gunicorn)
- Cloud SQL for Postgres
- Secret Manager (inject env vars)
- Cloud Run Jobs for migrations + superuser
- Cloud Logging for troubleshooting
- Cloud Storage for Django media (`django-storages`)

Your Django expects standard DB env vars (not `DATABASE_URL`):
- `DB_ENGINE`
- `DB_HOST`
- `DB_PORT`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD`

## 0) What you should see after deployment
- Visiting the Cloud Run service URL at `/` returns HTTP 200 and includes:
  - `It worked Khalid.`

## 1) One-time setup (your machine)
1. Login:
   ```bash
   gcloud auth login
   gcloud auth application-default login
   ```
2. Set the project:
   ```bash
   PROJECT_ID="academic-moon-466815-n6"
   gcloud config set project "$PROJECT_ID"
   ```

## 2) Enable required APIs
Run:
```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  logging.googleapis.com \
  storage-api.googleapis.com
```

## 3) Define names (edit defaults)
```bash
REGION="asia-south2"

SERVICE_NAME="django-todo-app"

# Cloud SQL
SQL_INSTANCE_NAME="django-todo-app-postgres"
SQL_DB_NAME="tododb"
SQL_DB_USER="todoappuser"

# Artifact Registry (Docker repo)
AR_REPO="django-todo-app-repo"
IMAGE_NAME="django-todo-app"
IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}"

# GCS bucket for media
# IMPORTANT: this must be the bucket name ONLY (NO `gs://` prefix).
GCS_BUCKET_NAME="your-unique-gcs-bucket-name"
```

Cloud SQL socket host used by this guide:
```bash
INSTANCE_CONNECTION_NAME="${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}"
DB_HOST="/cloudsql/${INSTANCE_CONNECTION_NAME}"
DB_PORT="5432"
```

## 4) Create Cloud SQL (Postgres)
### 4.1) Create instance
Your project may be **ENTERPRISE_PLUS**, where `db-f1-micro` can fail. Use a supported tier (example below):
```bash
gcloud sql instances create "$SQL_INSTANCE_NAME" \
  --database-version=POSTGRES_16 \
  --tier=db-perf-optimized-N-2 \
  --region="$REGION" \
  --quiet
```

If you want to check available tiers:
```bash
gcloud sql tiers list --show-edition --limit=50
```

### 4.2) Create database + user
```bash
gcloud sql databases create "$SQL_DB_NAME" --instance="$SQL_INSTANCE_NAME"
```

Set a DB password (important for zsh: use single quotes if it contains `!`):
```bash
DB_PASSWORD_VALUE='PUT_A_STRONG_PASSWORD_HERE'
```

Create the user:
```bash
gcloud sql users create "$SQL_DB_USER" \
  --instance="$SQL_INSTANCE_NAME" \
  --password="$DB_PASSWORD_VALUE"
```

## 5) Create secrets (Secret Manager)
Cloud Run will inject secrets as environment variables.

### 5.1) SECRET_KEY
```bash
SECRET_KEY_VALUE='PUT_A_SECURE_SECRET_KEY_HERE'

printf "%s" "$SECRET_KEY_VALUE" > /tmp/secret_key.txt
gcloud secrets create "django-todo-secret-key" \
  --replication-policy="automatic" \
  --data-file=/tmp/secret_key.txt
rm /tmp/secret_key.txt
```

### 5.2) DB_PASSWORD
```bash
printf "%s" "$DB_PASSWORD_VALUE" > /tmp/db_password.txt
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 5.3) Superuser password secret (recommended)
This app has a command to create a superuser if missing.

```bash
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"
DJANGO_SUPERUSER_PASSWORD_VALUE='PUT_A_SUPERUSER_PASSWORD_HERE'

printf "%s" "$DJANGO_SUPERUSER_PASSWORD_VALUE" > /tmp/superuser_password.txt
gcloud secrets create "django-todo-superuser-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/superuser_password.txt
rm /tmp/superuser_password.txt
```

## 6) Create GCS bucket (for Django media)
Create the bucket:
```bash
gsutil mb -p "$PROJECT_ID" -l "$REGION" "gs://${GCS_BUCKET_NAME}"
```

## 7) Create Artifact Registry repository
```bash
gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION" \
  --quiet
```

## 8) Build + push Docker image
Build:
```bash
cd "/Users/khalid/Documents/projects/d3v/Google Cloud Practice/Todo Django App"

gcloud builds submit \
  --tag "${IMAGE_URI}:latest" \
  .
```

## 9) Service Account (IAM) for Cloud Run (simplified)
Cloud Run runs under a service account. That account must have permissions to:
- read secrets (Secret Manager)
- connect to Cloud SQL (Cloud SQL socket/integration)
- write media to GCS bucket

### Recommended: create your own service account
```bash
RUN_SA_NAME="django-todo-run-sa"
RUN_SA_EMAIL="${RUN_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$RUN_SA_NAME" \
  --display-name="$RUN_SA_NAME"

gcloud secrets add-iam-policy-binding "django-todo-secret-key" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-db-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-superuser-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/cloudsql.client"

# IMPORTANT: GCS_BUCKET_NAME must be a bucket NAME only (example: "django-todo-app"),
# not a full "gs://..." URL and not empty.
gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

### If you skip creating a service account
- Cloud Run will use the project’s default service account.
- If that default account does NOT have the roles above, your Cloud Run service and jobs will fail with errors like:
  - permission denied for secrets
  - Cloud SQL connection/auth failures
  - GCS permission errors

If you skip Step 9, you must grant the same roles to the Cloud Run default service account.

## 10) Deploy Cloud Run service (CLI)
```bash
gcloud run deploy "$SERVICE_NAME" \
  --image "${IMAGE_URI}:latest" \
  --region "$REGION" \
  --allow-unauthenticated \
  --service-account "$RUN_SA_EMAIL" \
  --add-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest"
```

## 11) Run migrations (Cloud Run Job)
Migrations must run against the same DB.
```bash
gcloud run jobs deploy "django-todo-migrate" \
  --image "${IMAGE_URI}:latest" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest" \
  --command python \
  --args manage.py,migrate \
  --execute-now
```

## 12) Create Django superuser (Cloud Run Job)
Before running this job, set:
```bash
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"
```

```bash
gcloud run jobs deploy "django-todo-superuser" \
  --image "${IMAGE_URI}:latest" \
  --region "$REGION" \
  --service-account "$RUN_SA_EMAIL" \
  --set-cloudsql-instances "$INSTANCE_CONNECTION_NAME" \
  --set-env-vars "DEBUG=false,DB_ENGINE=postgres,DB_HOST=${DB_HOST},DB_PORT=${DB_PORT},DB_NAME=${SQL_DB_NAME},DB_USER=${SQL_DB_USER},GS_BUCKET_NAME=${GCS_BUCKET_NAME},ALLOWED_HOSTS=*,DJANGO_SUPERUSER_USERNAME=${DJANGO_SUPERUSER_USERNAME},DJANGO_SUPERUSER_EMAIL=${DJANGO_SUPERUSER_EMAIL}" \
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest,DJANGO_SUPERUSER_PASSWORD=django-todo-superuser-password:latest" \
  --command python \
  --args manage.py,create_superuser_if_missing \
  --execute-now
```

## 13) Cloud Logging (how to debug quickly)
View last logs:
```bash
gcloud run services logs read "$SERVICE_NAME" --region "$REGION" --limit 50
```

Common causes of `503` / “service unavailable”:
- missing env var or missing secret mapping
- Cloud SQL connection failure
- migrations not run yet

## 14) Verify the app
Service URL:
```bash
gcloud run services describe "$SERVICE_NAME" --region "$REGION" --format='value(status.url)'
```

Test home page:
```bash
curl -i "https://$SERVICE_URL/"
```

You should see `It worked Khalid.` in the HTML.

## 15) Local development env example
This repo includes `.env.example` (SQLite). For your production DB, follow the DB_* + secrets approach above.

## 16) Cleanup (optional, destructive)
Order (data loss):
1. Delete Cloud Run service and jobs
2. Delete Cloud SQL instance
3. Delete Artifact Registry repo
4. Delete GCS bucket
5. Delete secrets

# Django on Cloud Run (Docker) + Cloud SQL (Postgres) + Secrets + Cloud Logging + GCS

This is a clean, first-time, professional guide to deploy your Django app to Google Cloud using:
- Cloud Run
- Cloud SQL for Postgres
- Secret Manager (inject secrets as environment variables)
- Cloud Logging (use stdout/stderr)
- Cloud Storage (Django media via `django-storages`)

This guide is written for people who are new to Django deployment on Cloud Run and who prefer using the Google Cloud Console.

## 1) What you will end up with
- Visiting your Cloud Run URL in a browser shows the home page:
  - `It worked Khalid.`
- Your API is available under `/api/` (authenticated endpoints).
- Database migrations run in a Cloud Run Job.
- A Django superuser is created in a Cloud Run Job if missing.

## 2) One-time prerequisites (your computer)
1. Install Google Cloud SDK.
2. Login:
   ```bash
   gcloud auth login
   gcloud auth application-default login
   ```
3. Set the project:
   ```bash
   PROJECT_ID="academic-moon-466815-n6"
   gcloud config set project "$PROJECT_ID"
   ```

## 3) Enable required Google APIs
In the Google Cloud Console: APIs & Services -> Enable APIs.

Or run this:
```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  logging.googleapis.com \
  storage-api.googleapis.com
```

## 4) Create database (Cloud SQL Postgres)
In the Console:
1. Go to Cloud SQL -> Create instance
2. Choose:
   - Engine: PostgreSQL
   - Region: choose `asia-south2` (or your preferred region)
   - Tier: keep the smallest option for testing
3. Create instance

Then create:
1. A database
2. A database user
3. A strong password (store it in Secret Manager next)

Using gcloud (optional):
```bash
REGION="asia-south2"
SQL_INSTANCE_NAME="django-todo-app-postgres"
SQL_DB_NAME="tododb"
SQL_DB_USER="todoappuser"

gcloud sql instances create "$SQL_INSTANCE_NAME" \
  --database-version=POSTGRES_16 \
  # Cloud SQL edition in some projects doesn't allow db-f1-micro.
  # For ENTERPRISE_PLUS, use a perf-optimized tier.
  --tier=db-perf-optimized-N-2 \
  --region="$REGION" \
  --quiet

gcloud sql databases create "$SQL_DB_NAME" --instance="$SQL_INSTANCE_NAME"
```

## 5) Create secrets (Secret Manager)
Cloud Run will inject these values as environment variables into your container.

### 5.1) Create `SECRET_KEY`
1. Go to Secret Manager -> Create secret
2. Name: `django-todo-secret-key`
3. Value: your Django secret key

gcloud (optional):
```bash
# NOTE (zsh): if your secret contains `!`, zsh may try history expansion and fail.
# Use single quotes (recommended) or escape `!` as `\!`.
SECRET_KEY_VALUE='PUT_A_SECURE_SECRET_KEY_HERE'
printf "%s" "$SECRET_KEY_VALUE" > /tmp/secret_key.txt
gcloud secrets create "django-todo-secret-key" \
  --replication-policy="automatic" \
  --data-file=/tmp/secret_key.txt
rm /tmp/secret_key.txt
```

### 5.2) Create `DB_PASSWORD`
1. Create secret name: `django-todo-db-password`
2. Value: the password of the Cloud SQL user you created

gcloud (optional):
```bash
DB_PASSWORD='PUT_A_STRONG_PASSWORD_HERE'
printf "%s" "$DB_PASSWORD" > /tmp/db_password.txt
gcloud secrets create "django-todo-db-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/db_password.txt
rm /tmp/db_password.txt
```

### 5.3) Create superuser password secret (optional but recommended)
This app includes a command that creates a superuser if missing.

1. Create secret name: `django-todo-superuser-password`
2. Value: superuser password you want

gcloud (optional):
```bash
DJANGO_SUPERUSER_USERNAME="khalid"
DJANGO_SUPERUSER_EMAIL="khalid@example.com"
DJANGO_SUPERUSER_PASSWORD_VALUE='PUT_A_STRONG_PASSWORD_HERE'

printf "%s" "$DJANGO_SUPERUSER_PASSWORD_VALUE" > /tmp/superuser_password.txt
gcloud secrets create "django-todo-superuser-password" \
  --replication-policy="automatic" \
  --data-file=/tmp/superuser_password.txt
rm /tmp/superuser_password.txt
```

## 6) Create GCS bucket (for Django media)
In the Console:
1. Go to Cloud Storage -> Buckets -> Create bucket
2. Bucket name: choose a unique value (example: `django-todo-app`)
3. Location: choose the same region if possible

Your Django uses `django-storages` when `GS_BUCKET_NAME` is set.

## 7) Create Artifact Registry repository (Docker images)
In the Console:
1. Artifact Registry -> Repositories -> Create repository
2. Format: Docker
3. Location: `asia-south2`
4. Name: `django-todo-app-repo` (example)

gcloud (optional):
```bash
REGION="asia-south2"
AR_REPO="django-todo-app-repo"

gcloud artifacts repositories create "$AR_REPO" \
  --repository-format=docker \
  --location="$REGION"
```

Your image URI will be:
```bash
IMAGE_URI="${REGION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/django-todo-app"
```

## 8) Build and push the Docker image
From your project folder (where `Dockerfile` is):
```bash
cd "/Users/khalid/Documents/projects/d3v/Google Cloud Practice/Todo Django App"

gcloud builds submit \
  --tag "${IMAGE_URI}:latest" \
  .
```

## 9) Prepare environment variables (standard Django style)
This guide uses standard Django Postgres settings via env vars:
- `DB_ENGINE`
- `DB_HOST`
- `DB_PORT`
- `DB_NAME`
- `DB_USER`
- `DB_PASSWORD` (from Secret Manager)

Example env style for Cloud Run:
```bash
DEBUG=false
DB_ENGINE=postgres
DB_HOST=/cloudsql/${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}
DB_PORT=5432
DB_NAME=tododb
DB_USER=todoappuser
GS_BUCKET_NAME=your-gcs-bucket-name
ALLOWED_HOSTS=*
SECRET_KEY=... (secret)
```

Your repository already contains `.env.example` for local development.

## 10) Create a dedicated service account (IAM)
This reduces permission confusion and keeps logs clearer.

1. Create a service account (example name): `django-todo-run-sa`
2. Grant roles to that service account:
   - `roles/secretmanager.secretAccessor` on:
     - `django-todo-secret-key`
     - `django-todo-db-password`
     - `django-todo-superuser-password` (superuser job only)
   - `roles/cloudsql.client` on the project
   - `roles/storage.objectAdmin` on the GCS bucket

gcloud (optional):
```bash
RUN_SA_NAME="django-todo-run-sa"
RUN_SA_EMAIL="${RUN_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$RUN_SA_NAME" \
  --display-name="$RUN_SA_NAME"

gcloud secrets add-iam-policy-binding "django-todo-secret-key" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-db-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-superuser-password" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud secrets add-iam-policy-binding "django-todo-csrf-trusted-origins" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/cloudsql.client"

gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

## 11) Create Cloud Run service
In the Console:
1. Cloud Run -> Create Service
2. Service name: `django-todo-app`
3. Region: `asia-south2`
4. Authentication:
   - Choose “Allow unauthenticated”
5. Container:
   - Choose image from Artifact Registry: `IMAGE_URI:latest`
6. In “Container, Networking, Security”:
   - Enable Cloud SQL integration and pick your Postgres instance
   - Add environment variables:
     - `DEBUG=false`
     - `DB_ENGINE=postgres`
     - `DB_HOST=/cloudsql/${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}`
     - `DB_PORT=5432`
     - `DB_NAME=tododb`
     - `DB_USER=todoappuser`
     - `GS_BUCKET_NAME=your-gcs-bucket-name`
     - `ALLOWED_HOSTS=*`
   - Add secrets as environment variables:
     - `SECRET_KEY` <- secret `django-todo-secret-key`
     - `DB_PASSWORD` <- secret `django-todo-db-password`
     - `CSRF_TRUSTED_ORIGINS` <- secret `django-todo-csrf-trusted-origins` (value = your service URL, e.g. `https://django-todo-app-….asia-south2.run.app` — update secret after you know the URL)
7. Set the service account to: `django-todo-run-sa`
8. Create service

Open the service URL and confirm `/` returns:
- HTTP 200
- HTML containing `It worked Khalid.`

## 12) Run migrations (Cloud Run Job)
In the Console:
1. Cloud Run -> Jobs -> Create Job
2. Job name: `django-todo-migrate`
3. Image:
   - `IMAGE_URI:latest`
4. Command:
   - `python`
5. Arguments:
   - `manage.py,migrate`
6. Region: same as your service
7. Service account: same as service (`django-todo-run-sa`)
8. Cloud SQL integration: select your Postgres instance
9. Set the same environment variables and the secrets:
   - `SECRET_KEY` from `django-todo-secret-key`
   - `DB_PASSWORD` from `django-todo-db-password`
   - `CSRF_TRUSTED_ORIGINS` from `django-todo-csrf-trusted-origins`
10. Click “Run”

## 13) Create superuser (Cloud Run Job)
This creates the admin user once (skips if it already exists).

In the Console:
1. Create another Job:
   - Job name: `django-todo-superuser`
2. Command:
   - `python`
3. Arguments:
   - `manage.py,create_superuser_if_missing`
4. Set environment variables:
   - same DB vars as migrations
   - `DJANGO_SUPERUSER_USERNAME=khalid`
   - `DJANGO_SUPERUSER_EMAIL=khalid@example.com`
5. Set secrets:
   - `SECRET_KEY` from `django-todo-secret-key`
   - `DB_PASSWORD` from `django-todo-db-password`
   - `DJANGO_SUPERUSER_PASSWORD` from `django-todo-superuser-password`
6. Run the job

## 14) Cloud Logging (debugging)
Cloud Run sends stdout/stderr to Cloud Logging automatically.

Console:
1. Cloud Logging -> Log Explorer
2. Filter:
   - `resource.type="cloud_run_revision"`
   - `service_name="django-todo-app"`

Quick command:
```bash
gcloud run services logs read "django-todo-app" --region "asia-south2" --limit 50
```

## 15) Common edge cases and how to fix them

### A) Permission denied for Secret Manager
Fix:
- Ensure service and job service account has:
  - `roles/secretmanager.secretAccessor`
for:
- `django-todo-secret-key`
- `django-todo-db-password`
- `django-todo-superuser-password` (superuser job only)

### B) Cloud SQL connection failures
Fix:
- Ensure Cloud SQL integration is enabled for both service and jobs
- Ensure service account has `roles/cloudsql.client`
- Ensure DB settings are correct:
  - `DB_HOST=/cloudsql/${PROJECT_ID}:${REGION}:${SQL_INSTANCE_NAME}`
  - `DB_USER` and `DB_PASSWORD` match Cloud SQL user you created

### C) Media uploads to GCS fail with 403
Fix:
- Ensure service account has bucket permissions for `GS_BUCKET_NAME`
- Example: `roles/storage.objectAdmin`

### D) App returns 503 on Cloud Run
Fix:
- Check service logs immediately
- Usually the container is failing at startup due to missing env vars/secrets or DB connection errors

## 16) What you should NOT do in production
- Do not mount secrets into `/app`
- Do not use passwords in plain `.env` files for Cloud Run
- Do not rely on `DATABASE_URL` for this guide; use standard `DB_*` variables

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
  # Cloud SQL editions can restrict small tiers; perf-optimized tier works reliably.
  --tier=db-perf-optimized-N-2 \
  --region="$REGION" \
  --quiet
```

### 5.2) Create a DB password and store it in Secret Manager

Set a password locally (you can paste a strong value):
```bash
# NOTE (zsh): if your secret contains `!`, zsh may try history expansion and fail.
DB_PASSWORD='PUT_A_STRONG_PASSWORD_HERE'
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
# NOTE (zsh): if your secret contains `!`, zsh may try history expansion and fail.
# Use single quotes (recommended) or escape `!` as `\!`.
SECRET_KEY_VALUE='PUT_A_SECURE_SECRET_KEY_HERE'
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
DJANGO_SUPERUSER_PASSWORD_VALUE='PUT_A_STRONG_PASSWORD_HERE'
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
gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
  --member="serviceAccount:${RUN_SA_EMAIL}" \
  --role="roles/storage.objectAdmin"
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
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest,CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:latest"
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
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest,CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:latest" \
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
  --set-secrets "SECRET_KEY=django-todo-secret-key:latest,DB_PASSWORD=django-todo-db-password:latest,DJANGO_SUPERUSER_PASSWORD=django-todo-superuser-password:latest,CSRF_TRUSTED_ORIGINS=django-todo-csrf-trusted-origins:latest" \
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
     gcloud storage buckets add-iam-policy-binding "gs://${GCS_BUCKET_NAME}" \
       --member="serviceAccount:${RUN_SA_EMAIL}" \
       --role="roles/storage.objectAdmin"
     ```

## 13) Cleanup (optional)

If you want to delete everything later, delete in this order:

1. Cloud Run service and jobs
2. Cloud SQL instance
3. Artifact Registry repo
4. GCS bucket (after you confirm you do not need files)
5. Secrets

