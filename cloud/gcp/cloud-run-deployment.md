## Prerequisites

1. Authenticated either via ADC or using Service account.
   - Service account and/or user credentials should have sufficient role permissions in order to deploy/build using cloudbuild.
   - For this example, the roles needed are:
     - [ ] Cloud Build Editor
     - [ ] Cloud Run Admin / invoker
   - Needed roles will vary depending on the app you are serving.

  
2. Creating your dockerfile
   - The contents of your dockerfile will solely depend on what your project needs and used.
   - In this example, it is a simple node.js server.
```dockerfile
# 1. BASE ENVIRONMENT SELECTION
# This instruction selects a pre-configured, minimal operating environment 
# containing the necessary runtime and system libraries. Using a 
# specialized "alpine" version ensures a smaller, more secure footprint.
FROM node:19-alpine

# 2. WORKSPACE INITIALIZATION
# This directive establishes the primary directory within the virtual 
# environment where all subsequent application operations will occur.
WORKDIR /usr/src/app

# 3. DEPENDENCY OPTIMIZATION
# These manifests are transferred independently. By isolating the 
# configuration of external libraries, the system can skip the 
# installation phase in future builds if the dependencies remain unchanged.
COPY package*.json tsconfig.json ./

# 4. EXTERNAL LIBRARY INSTALLATION
# This command executes the package manager to download and configure 
# all external code libraries required for the application to function.
RUN npm install

# 5. SOURCE CODE TRANSFER
# This step transfers the application's unique source code from the 
# development environment into the virtual workspace.
COPY . ./

# 6. COMPILATION PHASE
# This instruction converts the human-readable source code into a 
# specialized format optimized for execution by the runtime environment.
RUN npm run build

# 7. CONFIGURATION INJECTION
# These directives allow external metadata to be passed into the 
# environment during the creation process and stored for use while 
# the application is active.
ARG PROJECT_ID
ENV PROJECT_ID=$PROJECT_ID

# 8. EXECUTION PROTOCOL
# This defines the specific command that initializes the application 
# whenever the environment is activated.
CMD ["node", "dist/server.js"]
```


3. Building your cloudbuild file to building to deploying your app using one command.
```yaml
# This configuration file defines a sequenced automation for application delivery.
steps:
  # STEP 1: Building your Docker Image.
  # This step bundles the application source code and its required environment
  # into a single, immutable package. This ensures the software runs 
  # consistently across different computing environments.
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '--build-arg', 'PROJECT_ID=$PROJECT_ID', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>', '.']

  # STEP 2: Uploading your docker image to Artifact Registry
  # This step moves the completed application package from the local 
  # build environment to your selected artifact registry repo. 
  # This makes the package accessible for future retrieval and deployment.
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>']

  # STEP 3: ACTIVATION AND HOSTING
  # This step instructs the hosting service to retrieve the docker image
  # and start the application. It defines the physical location of the 
  # servers, allows public network access, and sets a specific 
  # identification variable for the application to use during runtime.
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args:
      [
        'run',
        'deploy',
        <CLOUD_RUN_SERVICE_NAME>, #ex: 'test-project'
        '--image',
        'us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>',
        '--region',
        '<REGION>',
        '--platform',
        'managed',
        '--allow-unauthenticated', # Grants access to users on the public internet
        '--set-env-vars',
        'PROJECT_ID=<GCP_PROJECT_ID>' # Injects a specific project name into the program
      ]

# RECORD KEEPING: 
# This final section explicitly tracks the specific application package 
# that was produced and stored during this automated sequence.
images: ['us-central1-docker.pkg.dev/$PROJECT_ID/<ARTIFACT_REGISTRY_FOLDER>/<DOCKER_IMAGE_NAME>:<TAG>']
```
