# ✅ Let’s Start — Your pipeline has only 2 parts

1.  **definitions** = the reusable step
2.  **pipelines** = when to run the step

So we’ll start with definitions.

---

## 🧩 PART 1 — definitions

This creates a reusable block called `build-and-deploy-service`.

```yaml
definitions:
  steps:
    - step: &build-and-deploy-service
```

👉 **Meaning:**
You are creating a step template named `build-and-deploy-service`.
Later you will call it using `<<: *build-and-deploy-service`.

### 🐳 The environment for this step

```yaml
image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
services:
    - docker
```

👉 **Meaning:**
*   Use a Google Cloud SDK image (so you get `gcloud` command)
*   Enable Docker engine inside the pipeline → because you will build Docker images

### 📦 script: The actual things the pipeline does

Everything inside the `script`: runs step-by-step.

#### 🔵 STEP 1 — Basic setup

```bash
SERVICE_NAME="ellume-core"
ENV_NAME=$(echo "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" | tr '[:upper:]' '[:lower:]')
```

👉 **Meaning:**
*   Name of the Cloud Run service = `ellume-core`
*   Convert environment → lowercase (Development → development, Production → production)

#### 🔵 STEP 2 — Validate environment

```bash
if [ "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" != "development" ] &&
    [ "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" != "production" ] &&
    [ "$BITBUCKET_DEPLOYMENT_ENVIRONMENT" != "staging" ]; then
  echo "Invalid DEPLOYMENT..."
  exit 1
fi
```

👉 **Meaning:**
Only allow development, staging, production.
If someone tries to deploy other environment → **STOP**.

#### 🔵 STEP 3 — Check if all required secrets exist

```bash
required_secrets=(
  "SA_BUILD_DEPLOY"
  "PROJECT_ID"
  "REGION"
  "SERVER_PORT"
  "JWT_SECRET"
  "ARTIFACT_REGISTRY_URL"
  "DATABASE_DRIVER"
  "DATABASE_DSN"
  "CLOUDSQL_INSTANCE"
)
```

👉 **Meaning:**
Pipeline needs these secrets. If any missing → deployment fails.

Then this loop checks one by one:

```bash
for secret in "${required_secrets[@]}"; do
  if [ -z "${!secret}" ]; then
    missing_secrets+=("$secret")
  fi
done
```

👉 **Meaning:**
*   `${!secret}` = get value of secret name (Example: `PROJECT_ID` → get its value)
*   If missing → add to `missing_secrets`.
*   If any missing → print them → exit.

#### 🔵 STEP 4 — Authenticate to Google Cloud

```bash
echo "${SA_BUILD_DEPLOY}" | base64 -d > ${HOME}/gcp-key.json
gcloud auth activate-service-account --key-file ${HOME}/gcp-key.json
gcloud config set project $PROJECT_ID
gcloud auth configure-docker $REGION-docker.pkg.dev --quiet
```

👉 **Meaning:**
1.  Decode service account key
2.  Login to Google Cloud
3.  Set the correct project
4.  Allow Docker to push images to Artifact Registry

Your pipeline now has access to GCP.

#### 🔵 STEP 5 — Build Docker image with 3 tags

```bash
BUILD_TIME=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
BUILD_TIMESTAMP=$(date -u +'%s')
```

👉 **Meaning:**
*   Build time in ISO format
*   Build time as UNIX timestamp

Now create 3 image tags:

```bash
LATEST_TAG="...:development-latest"
DATED_TAG="...:development-commitHash"
VERSION_TAG="...:development-timestamp"
```

👉 **Why 3 tags?**
*   `latest` → newest version
*   `commit` → helps rollback
*   `timestamp` → exact build

Now build:

```bash
docker build \
   --tag "$LATEST_TAG" \
   --tag "$DATED_TAG" \
   --tag "$VERSION_TAG" \
   --build-arg GITHUB_SHA="$BITBUCKET_COMMIT" \
   --build-arg GITHUB_REF="$BITBUCKET_BRANCH" \
   --build-arg BUILD_TIME="$BUILD_TIME" \
   --build-arg BUILD_ENVIRONMENT="$ENV_NAME" \
   .
```

👉 **Meaning:**
You build ONE image but tag it 3 different ways.
Also pass build info into Dockerfile.

#### 🔵 STEP 6 — Push images to Artifact Registry

```bash
docker push "$LATEST_TAG"
docker push "$DATED_TAG"
docker push "$VERSION_TAG"
```

👉 **Meaning:**
Upload all 3 versions to GCP registry.

#### 🔵 STEP 7 — Deploy to Cloud Run

```bash
DEPLOY_IMAGE="${ARTIFACT_REGISTRY_URL}/${SERVICE_NAME}:${ENV_NAME}-latest"
```

👉 **Meaning:**
Cloud Run always uses the latest image for that environment.

Then:

```bash
gcloud run deploy ellume-core \
   --image="$DEPLOY_IMAGE" \
   --region=$REGION \
   --platform=managed \
   --allow-unauthenticated \
   --port=$SERVER_PORT \
   --project=$PROJECT_ID \
   --add-cloudsql-instances=$CLOUDSQL_INSTANCE \
   --set-env-vars=ENVIRONMENT=$ENV_NAME,...
```

👉 **Meaning:**
You tell Cloud Run:
*   which image to deploy
*   which GCP region
*   which port to expose
*   which SQL instance to connect
*   which environment variables to set

Cloud Run now:
1.  pulls image
2.  replaces previous version
3.  publishes new URL

#### 🔵 STEP 8 — Cleanup

```bash
rm -f ${HOME}/gcp-key.json
```

👉 **Meaning:**
Delete the service account key for security.

---

## 🟢 PART 2 — pipelines

```yaml
pipelines:
  branches:
    new_dev:
      - step:
          <<: *build-and-deploy-service
          deployment: Development
```

👉 **Meaning:**
Whenever you push code to `new_dev` branch:
1.  Use the reusable step (`<<: *build-and-deploy-service`)
2.  Set environment = `Development`

And then the whole script runs for development.

---

## 🎉 Super Simple Summary With Code Sense

Here is what your pipeline does in order, using the code you pasted:

1.  Set `SERVICE_NAME` + `ENV_NAME`
2.  Check environment is valid
3.  Check required secrets exist
4.  Login to Google Cloud
5.  Build Docker image with 3 tags
6.  Push all 3 tags to Artifact Registry
7.  Deploy the latest tag to Cloud Run
8.  Clean temporary GCP key
