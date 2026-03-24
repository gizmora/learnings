# GCP Setup

Steps to take before deploying to Cloud Run.

---

## 1. Authentication

### Login as yourself
```bash
gcloud auth login
gcloud config set project <GCP_PROJECT_ID>
```

### Login as a service account
```bash
gcloud auth activate-service-account <service-account-name>@<project-id>.iam.gserviceaccount.com --key-file=<service-account-key>.json
```

---

## 2. Verify Cloud Build Service Account Permissions

Cloud Build runs deployments using its own service account (`<PROJECT_NUMBER>@cloudbuild.gserviceaccount.com`). It needs the following roles to deploy to Cloud Run:

| Role | Purpose |
|------|---------|
| `roles/run.admin` | Deploy and manage Cloud Run services |
| `roles/iam.serviceAccountUser` | Act as the Cloud Run runtime service account |

> This step requires sufficient IAM permissions. If you don't have them, ask your GCP project admin to grant these roles to the Cloud Build service account.

### Option A: GCP Console (GUI)

1. Go to **IAM & Admin → IAM** in the GCP Console
2. Find the Cloud Build service account — it looks like `<PROJECT_NUMBER>@cloudbuild.gserviceaccount.com`
3. Click the pencil icon to edit its roles
4. Add **Cloud Run Admin** and **Service Account User**
5. Click **Save**

### Option B: CLI

First, find your project number:
```bash
gcloud projects describe <GCP_PROJECT_ID> --format="value(projectNumber)"
```

Then grant the roles:
```bash
gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
  --member="serviceAccount:<PROJECT_NUMBER>@cloudbuild.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding <GCP_PROJECT_ID> \
  --member="serviceAccount:<PROJECT_NUMBER>@cloudbuild.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

---

## 3. Create an Artifact Registry Repository

The repository must exist before you can push Docker images to it.

### Option A: GCP Console (GUI)

1. Go to **Artifact Registry** in the GCP Console
2. Click **Create Repository**
3. Set the format to **Docker**
4. Choose a name and region, then click **Create**

### Option B: CLI

```bash
gcloud artifacts repositories create <ARTIFACT_REGISTRY_FOLDER> \
  --repository-format=docker \
  --location=<REGION> \
  --project=<GCP_PROJECT_ID>
```

Verify it was created:
```bash
gcloud artifacts repositories list --location=<REGION>
```
