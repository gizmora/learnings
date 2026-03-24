# Deploying to Cloud Run with Cloud Build

This guide walks you through deploying a Node.js app to Google Cloud Run using a single `gcloud builds submit` command.

> Before continuing, make sure you've completed the steps in [gcp-setup.md](gcp-setup.md) — authentication, Cloud Build permissions, and creating your Artifact Registry repository.

---

## Prerequisites

### 1. Create a Dockerfile

The Dockerfile tells Docker how to build and run your app. Here's an example for a Node.js TypeScript project:

```dockerfile
# Use a lightweight Node.js image as the base
FROM node:19-alpine

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy dependency files first (this lets Docker cache the npm install step)
COPY package*.json tsconfig.json ./

# Install dependencies
RUN npm install

# Copy the rest of your source code
COPY . ./

# Build the TypeScript source into JavaScript
RUN npm run build

# Accept the GCP project ID as a build argument and store it as an env variable
ARG PROJECT_ID
ENV PROJECT_ID=$PROJECT_ID

# Cloud Run injects a PORT env variable — your app must listen on it
EXPOSE 8080

# Start the app
CMD ["node", "dist/server.js"]
```

> **Why copy `package.json` before the source code?**
> Docker builds in layers and caches each step. Copying dependency files first means Docker can skip `npm install` on future builds if your dependencies haven't changed — saving time.

> **PORT:** Cloud Run sets a `PORT` environment variable (default `8080`) and routes traffic to it. Make sure your app reads from `process.env.PORT` rather than hardcoding a port number.

### 2. Create a `.dockerignore` File

Prevents unnecessary files from being sent to Cloud Build, keeping builds fast.

```
node_modules
dist
.git
.env
*.log
```

---

### 3. Create a `cloudbuild.yaml` File

This file defines the build-and-deploy pipeline that Cloud Build will run. Replace all `<PLACEHOLDER>` values with your own.

```yaml
steps:
  # Step 1: Build the Docker image
  # Packages your app + environment into a container image.
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '--build-arg'
      - 'PROJECT_ID=$PROJECT_ID'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>'
      - '.'

  # Step 2: Push the image to Artifact Registry
  # Uploads the image so Cloud Run can pull it during deployment.
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>'

  # Step 3: Deploy to Cloud Run
  # Tells Cloud Run to pull the image and start serving it.
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - '<CLOUD_RUN_SERVICE_NAME>'       # e.g. 'my-app'
      - '--image'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>'
      - '--region'
      - '<REGION>'                        # e.g. 'us-central1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'         # Makes the service publicly accessible
      - '--set-env-vars'
      - 'PROJECT_ID=<GCP_PROJECT_ID>'     # Passes your project ID into the running container

# Tells Cloud Build to track this image in its build artifacts
images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>'
```

---

## Placeholders Reference

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<ARTIFACT_REGISTRY_FOLDER>` | Your Artifact Registry repository name | `my-repo` |
| `<DOCKER_IMAGE_NAME>` | Name to give your Docker image | `my-app` |
| `<TAG>` | Version tag for the image | `latest` or `v1.0` |
| `<CLOUD_RUN_SERVICE_NAME>` | Name of your Cloud Run service | `my-app` |
| `<REGION>` | GCP region to deploy to | `us-central1` |
| `<GCP_PROJECT_ID>` | Your GCP project ID | `my-project-123` |

---

## Running the Deployment

Once your files are in place, trigger the build with:

```bash
gcloud builds submit --config cloudbuild.yaml
```

This single command will:
1. Build your Docker image
2. Push it to Artifact Registry
3. Deploy it to Cloud Run
