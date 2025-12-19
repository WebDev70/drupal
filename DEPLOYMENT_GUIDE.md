# Deploying a Dockerized Drupal Application to GKE via Google Cloud Build (Granular Guide)

This guide provides a detailed, step-by-step process for setting up a fully automated CI/CD pipeline. We will take a Dockerized Drupal application, host it on GitHub, and create a workflow using Google Cloud Build that automatically builds and deploys it to the Google Kubernetes Engine (GKE).

## What This Guide Will Build

We are creating an automated pipeline with the following flow:

1.  You push code changes from your local computer to your GitHub repository.
2.  A **Cloud Build Trigger** detects this push.
3.  The trigger starts a **Cloud Build** job.
4.  Cloud Build follows the instructions in your `cloudbuild.yaml` file to:
    *   Build your Docker images.
    *   Push them to a private **Artifact Registry**.
    *   Deploy the new images to your **GKE Cluster**.
5.  GKE runs your application, which connects securely to a **Cloud SQL** database.

## Core Concepts

*   **Docker:** A tool to package applications and their dependencies into isolated "containers". This ensures your application runs the same way everywhere.
*   **Kubernetes (and GKE):** A powerful system for managing containerized applications. It handles running, scaling, and networking them. GKE is Google's managed version of Kubernetes, which handles the complex setup for you.
*   **CI/CD (and Google Cloud Build):** CI/CD stands for Continuous Integration/Continuous Deployment. It's the practice of automating your build and deployment process. Cloud Build is Google's managed CI/CD service that can automatically build, test, and deploy your code when it changes.

## Prerequisites

*   A Google Cloud Project with a billing account enabled.
*   The `gcloud` command-line tool. [Official Installation Guide](https://cloud.google.com/sdk/docs/install).
*   The `kubectl` command-line tool. It is installed with `gcloud`: `gcloud components install kubectl`.
*   Docker Desktop. [Official Installation Guide](https://docs.docker.com/get-docker/).
*   Your Drupal project, including the Dockerfiles, pushed to a GitHub repository.

---

## Step 1: Set Up Google Cloud Resources

This is the foundational step where we create all the necessary infrastructure in your Google Cloud project.

### 1.1. Configure Environment Variables
The `export` command sets a variable in your terminal session. This saves you from having to repeatedly type these values. These variables are temporary and only exist in the terminal window you run them in.

Open your computer's terminal and run the following block. It will define names and settings for the resources we're about to create.

```bash
# This command gets your currently configured Google Cloud Project ID.
export PROJECT_ID="$(gcloud config get-value project)"

# This is the geographical region where your resources will be created.
export REGION="us-central1"
export ZONE="us-central1-a"

# These are the public names for the resources we will create.
export GKE_CLUSTER_NAME="drupal-cluster"
export AR_REPO_NAME="drupal-repo"
export SQL_INSTANCE_NAME="drupal-db-instance"
export DB_NAME="drupal_db"
export DB_USER="drupal_user"
```

### 1.2. Run Cloud Setup Commands
Run these commands one by one in your terminal.

**1. Enable Necessary APIs**
APIs (Application Programming Interfaces) are like control panels that let us manage services programmatically. This command switches on the controls for GKE, Artifact Registry, Cloud SQL, and Secret Manager so we can create and manage them.
```bash
gcloud services enable container.googleapis.com artifactregistry.googleapis.com sqladmin.googleapis.com secretmanager.googleapis.com cloudbuild.googleapis.com
```

**2. Create an Artifact Registry Repository**
Your Docker images need to be stored somewhere your cloud environment can access them. Artifact Registry is a private and secure place to store these images, like a private Docker Hub.
```bash
gcloud artifacts repositories create ${AR_REPO_NAME} --repository-format=docker --location=${REGION}
```

**3. Create a Cloud SQL Instance**
For a real application, you want a reliable, managed database. Cloud SQL handles backups, updates, and scaling for you. This command creates a small but capable MySQL instance.
*(This step can take 5-10 minutes. You will see it processing in your terminal.)*
```bash
gcloud sql instances create ${SQL_INSTANCE_NAME} --database-version=MYSQL_8_0 --region=${REGION} --cpu=2 --memory=4GB
```

**4. Create the Drupal Database and User**
Now, within the SQL instance, we create the specific database for Drupal and a user account that Drupal will use to access it.
```bash
gcloud sql databases create ${DB_NAME} --instance=${SQL_INSTANCE_NAME}

# IMPORTANT: This next command will prompt you to set a password for the new user.
# Choose a strong password, and save it somewhere safe (like a password manager).
# You will need this password in the next step.
gcloud sql users create ${DB_USER} --host=% --instance=${SQL_INSTANCE_NAME}
```

**5. Create a GKE Cluster**
To stay within the default quotas of new Google Cloud projects, we will create a smaller, single-node *zonal* cluster instead of a larger regional one. This is sufficient for development and testing. We also enable Workload Identity, a secure way for applications inside GKE to access other Google Cloud services.
*(This step can take 5-10 minutes.)*
```bash
gcloud container clusters create ${GKE_CLUSTER_NAME} --zone=${ZONE} --num-nodes=1 --workload-pool=${PROJECT_ID}.svc.id.goog
```

---

## Step 2: Store Your Database Password Securely

Never put passwords or keys directly in your code or configuration files. We will use Google Secret Manager, a service designed to store and manage sensitive data.

1.  **Create a secret container:** This command creates a logical container for your secret versions.
    ```bash
    gcloud secrets create DB_PASSWORD --replication-policy="automatic"
    ```
2.  **Add your password as a secret version.** This command takes your password and stores it as the first version of the secret you just created.
    **Replace `your-super-secret-password` with the actual database password you saved.**
    ```bash
    echo -n "Basing49" | gcloud secrets versions add DB_PASSWORD --data-file=-
    ```

---

## Step 3: Create Kubernetes Manifests

These YAML files are the blueprints for your application in Kubernetes. They should be in a `k8s/` directory.

### `k8s/persistent-volume-claim.yml`
Drupal needs to store uploaded files (images, documents) persistently. Since Kubernetes pods can be deleted and recreated at any time, we need to claim a piece of network-attached storage that exists independently of the pod. This manifest requests 10GB of standard storage.
```yaml
# ... (File content is correct from previous steps)
```

### `k8s/deployment.yml`
This is the most important manifest. It defines a "Deployment," which manages the lifecycle of your application "Pods". A Pod is the smallest unit in Kubernetes and holds your running container(s).
*   **`__IMAGE_URL__`, `__IMAGE_TAG__`, `__INSTANCE_CONNECTION_NAME__`:** These are placeholders that our Cloud Build script will find and replace with real values during the deployment process.
```yaml
# ... (File content is correct from previous steps, including placeholders)
```

### `k8s/service.yml`
A "Service" in Kubernetes provides a stable network endpoint (like an IP address and port) to access one or more Pods. This is necessary because Pods can be replaced, getting new IP addresses. The Service IP address remains constant.
```yaml
# ... (File content is correct from previous steps)
```

---

## Step 4: Create the Cloud Build Configuration

The `cloudbuild.yaml` file tells Cloud Build the exact steps to run. It's a sequence of commands, where each step runs inside a specific container "builder".

### `cloudbuild.yaml`
```yaml
# cloudbuild.yaml

# 'steps' is a list of commands Cloud Build will execute in order.
steps:
  # Each step runs in a 'builder', which is a Docker container with common tools.
  # This step uses the 'docker' builder to build our Drupal image.
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_AR_REPO_NAME}/drupal:$SHORT_SHA', '-f', 'Dockerfile.drupal', '.']

  # Build the Adminer image.
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_AR_REPO_NAME}/adminer:$SHORT_SHA', '-f', 'Dockerfile.adminer', '.']

  # This step configures the 'kubectl' command-line tool to connect to our GKE cluster.
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - 'gcloud config set project $PROJECT_ID && gcloud container clusters get-credentials ${_GKE_CLUSTER_NAME} --zone ${_ZONE}'

  # This step uses the 'sed' command to find and replace the placeholders in our deployment manifest.
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        sed -i "s|__IMAGE_URL__|${_REGION}-docker.pkg.dev/$PROJECT_ID/${_AR_REPO_NAME}|g" k8s/deployment.yml
        sed -i "s|__IMAGE_TAG__|${SHORT_SHA}|g" k8s/deployment.yml
        sed -i "s|__INSTANCE_CONNECTION_NAME__|$PROJECT_ID:${_REGION}:${_SQL_INSTANCE_NAME}|g" k8s/deployment.yml

  # This step fetches the database password from Secret Manager and saves it to a temporary file.
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args: ['-c', 'gcloud config set project $PROJECT_ID && gcloud secrets versions access latest --secret=DB_PASSWORD > /workspace/db-password']
    id: 'GET_DB_PASSWORD'

  # This step creates a Kubernetes Secret from the password file fetched in the previous step.
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        kubectl create secret generic drupal-secrets \
          --from-file=DB_PASSWORD=/workspace/db-password \
          --dry-run=client -o yaml | kubectl apply -f -
    waitFor: ['GET_DB_PASSWORD'] # Ensures the password has been fetched first

  # Finally, this step applies all our manifests to the GKE cluster, creating or updating our resources.
  - name: 'gcr.io/cloud-builders/gcloud'
    entrypoint: 'bash'
    args:
      - '-c'
      - 'kubectl apply -f k8s/'

# After all steps are successful, Cloud Build will push these images to Artifact Registry.
images:
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_AR_REPO_NAME}/drupal:$SHORT_SHA'
  - '${_REGION}-docker.pkg.dev/$PROJECT_ID/${_AR_REPO_NAME}/adminer:$SHORT_SHA'

# These are user-defined variables that can be passed in from the Trigger. We set defaults here.
substitutions:
  _REGION: 'us-central1'
  _ZONE: 'us-central1-a'
  _GKE_CLUSTER_NAME: 'drupal-cluster'
  _AR_REPO_NAME: 'drupal-repo'
  _SQL_INSTANCE_NAME: 'drupal-db-instance'

# This option sends logs directly to Cloud Logging for better viewing.
options:
  logging: CLOUD_LOGGING_ONLY
```

---

## Step 5: Create and Configure a Dedicated Build Service Account

To ensure our build has an identity with the correct permissions, and to avoid issues where default accounts don't exist, we will manually create a dedicated service account for our Cloud Build pipeline. This is a security best practice.

1.  **Create the Service Account:**
    This command creates a new service account named `cloud-build-deployer` in your project.
    ```bash
    gcloud iam service-accounts create cloud-build-deployer \
      --display-name="Cloud Build Deployer SA" \
      --project=$PROJECT_ID
    ```

2.  **Grant Permissions to the New Service Account:**
    Now, we grant the roles this new account needs to perform the deployment.
    ```bash
    export SERVICE_ACCOUNT_EMAIL="cloud-build-deployer@${PROJECT_ID}.iam.gserviceaccount.com"

    # Grant GKE Developer role
    gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/container.developer"
    # Grant Artifact Registry Writer role
    gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/artifactregistry.writer"
    # Grant Cloud SQL Client role
    gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/cloudsql.client"
    # Grant Secret Manager Accessor role
    gcloud projects add-iam-policy-binding $PROJECT_ID --member="serviceAccount:${SERVICE_ACCOUNT_EMAIL}" --role="roles/secretmanager.secretAccessor"
    ```

3.  **Allow Your User to Select This Service Account:**
    Finally, you must grant your own user account permission to "act as" this new service account. This is required to select it in the Cloud Build trigger UI.
    *   First, find your user email: `gcloud auth list`
    *   Then, run the command below, replacing `YOUR_USER_EMAIL` with your email.
    ```bash
    export YOUR_USER_EMAIL=your-email@example.com
    
    gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
        --project=$PROJECT_ID \
        --member="user:${YOUR_USER_EMAIL}" \
        --role="roles/iam.serviceAccountUser"
    ```

---

## Step 6: Create the Cloud Build Trigger

This trigger connects GitHub to Cloud Build.

1.  Open the Cloud Console to **Cloud Build** -> **Triggers**.
2.  Click **Create trigger**.
3.  **Name:** `deploy-to-gke`.
4.  **Region:** `us-central1` (or your chosen region).
5.  **Event:** Select **Push to a branch**.
6.  **Source:** Connect and select your GitHub repository.
7.  **Branch:** `^main$`.
8.  **Configuration:** Select **Cloud Build configuration file** and set the location to `cloudbuild.yaml`.
9.  **Advanced (IMPORTANT):**
    *   Expand the **Advanced** section.
    *   Find the **Service account** field. Click the dropdown and select your new **`cloud-build-deployer@...`** service account.
10. Click **Create**.

***
### **Troubleshooting Note: Wrong Project ID in Errors**
If your build fails with an error mentioning the wrong project ID, it means your trigger was created while your Cloud Console was set to the wrong project. To fix this, delete the trigger and re-create it, ensuring you are in the correct project first.
***

---

## Step 7: Trigger the Deployment

Commit your `cloudbuild.yaml` file and the updated guide to your repository. This push will be detected by the trigger and start your first automated deployment.

```bash
git add cloudbuild.yaml DEPLOYMENT_GUIDE.md
git commit -m "feat: Implement Cloud Build pipeline"
git push origin main
```

---

## Step 8: Monitor and Access Your Application

1.  **Monitor:** In the Google Cloud Console, go to **Cloud Build** -> **History**. You will see your build running (it will appear blue). If it succeeds, it will turn green; if it fails, it will turn red. Click on the build to see detailed logs for each step.
2.  **Get IP Address:** After the build succeeds, it can take a few minutes for Google to provision the public IP addresses. To watch for them, run this command:
    ```bash
    kubectl get services --watch
    ```
    Wait for an `EXTERNAL-IP` to appear for the `drupal-service`. If it stays `<pending>` for more than 10 minutes, there might be an issue. You can debug by running `kubectl describe service drupal-service`.
3.  **Access:** Open a web browser and navigate to the external IP address of your `drupal-service`. You should see the Drupal installation screen!

