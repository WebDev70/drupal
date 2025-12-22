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
This manifest now defines a single Deployment for a Pod containing three containers: your Drupal application, the Cloud SQL Auth Proxy sidecar, and the Adminer sidecar.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drupal
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drupal
  template:
    metadata:
      labels:
        app: drupal
    spec:
      securityContext:
        fsGroup: 33
      containers:
        # Drupal Application Container
        - name: drupal
          image: __IMAGE_URL__/drupal:__IMAGE_TAG__
          ports:
            - containerPort: 80
          env:
            - name: DB_HOST
              value: "127.0.0.1"
            - name: DB_PORT
              value: "3306"
            - name: DB_NAME
              value: "drupal_db"
            - name: DB_USER
              value: "drupal_user"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: drupal-secrets
                  key: DB_PASSWORD
          volumeMounts:
            - name: drupal-persistent-storage
              mountPath: /var/www/html/sites/default/files
          startupProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30

        # Cloud SQL Auth Proxy Container (Sidecar)
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
          args:
            - "--structured-logs"
            - "--port=3306"
            - "__INSTANCE_CONNECTION_NAME__"
          securityContext:
            runAsUser: 0

        # Adminer Container (Sidecar)
        - name: adminer
          image: __IMAGE_URL__/adminer:__IMAGE_TAG__
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: adminer-sessions
              mountPath: /tmp

      volumes:
        - name: drupal-persistent-storage
          persistentVolumeClaim:
            claimName: drupal-pvc
        - name: adminer-sessions
          emptyDir: {}
```

### `k8s/service.yml`
This file is now simpler and only defines the service to expose the Drupal application.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: drupal-service
spec:
  type: LoadBalancer
  selector:
    app: drupal
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

---

## Step 4: Create the Cloud Build Configuration
(This step remains unchanged)

### `cloudbuild.yaml`
```yaml
# cloudbuild.yaml
# ... (File content is correct from previous steps)
```
---

## Step 5: Create and Configure Service Accounts
(This step remains unchanged)

---

## Step 6: Create the Cloud Build Trigger
(This step remains unchanged)

***
### **Troubleshooting Note: Wrong Project ID in Errors**
(This note remains unchanged)
***
### **Troubleshooting Database Access ("Access Denied")**
(This note remains unchanged)
***

---

## Step 7: Trigger the Deployment
(This step remains unchanged)

---

## Step 8: Monitor and Access Your Application

1.  **Monitor:** Monitor your build in the Cloud Build History page.

2.  **Get Drupal's Public IP Address:** Run `kubectl get services --watch` and wait for an `EXTERNAL-IP` to appear for the `drupal-service`.

3.  **Access Your Drupal Site:** Navigate to the external IP address in your browser to complete the Drupal installation.

4.  **Accessing Adminer Securely (New Method):**
    Since Adminer now runs inside the Drupal pod and is not exposed by a service, we must connect to it by port-forwarding directly to the pod.

    *   **First, get your Drupal pod's name:**
        ```bash
        kubectl get pods -l app=drupal
        ```
        The output will look like `drupal-86c8b4bdbb-5t9k5`. Copy this name.

    *   **Create the secure tunnel:** Run the following command, replacing `<YOUR_POD_NAME>` with the name you just copied.
        ```bash
        kubectl port-forward pod/<YOUR_POD_NAME> 8080:8080
        ```
    *   **Connect to Adminer:** While the command is running, open a browser and go to `http://localhost:8080`.

    *   **Log In to Adminer:** Use the following credentials to log in.
        *   **System:** `MySQL`
        *   **Server:** `127.0.0.1` (This connects to the Cloud SQL Proxy running in the same pod)
        *   **Username:** `drupal_user`
        *   **Password:** The password you stored in Secret Manager.
        *   **Database:** `drupal_db`
    *   To stop the tunnel, go back to the terminal and press `Ctrl+C`.

