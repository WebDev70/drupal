# Project: Drupal on Google Kubernetes Engine

## Project Overview

This project contains the complete infrastructure-as-code configuration to run a Drupal CMS application, both locally via Docker Compose and deployed in a production-like environment on Google Kubernetes Engine (GKE).

The architecture includes:
*   **Application:** A containerized Drupal application.
*   **Database:** A MariaDB database for Drupal. For cloud deployment, this is a managed Cloud SQL instance.
*   **Local Development:** A `docker-compose.yml` file orchestrates the local environment, including an Adminer service for database management.
*   **Cloud Deployment:** The deployment to GKE is fully automated via a CI/CD pipeline using Google Cloud Build. Pushing to the `main` GitHub branch triggers a build that builds container images, pushes them to Artifact Registry, and deploys to the GKE cluster.
*   **Configuration as Code:** All aspects of the deployment, from the Kubernetes objects (`k8s/` directory) to the CI/CD pipeline (`cloudbuild.yaml`), are defined in code within this repository.

## Building and Running

### Local Development

The local environment is managed by Docker Compose.

1.  **Prerequisites:** Docker and Docker Compose must be installed.
2.  **Build and Start:** To build the images and start all services (Drupal, MariaDB, Adminer) in the background, run:
    ```bash
    docker-compose up --build -d
    ```
3.  **Accessing Services:**
    *   **Drupal:** [http://localhost:8095](http://localhost:8095)
    *   **Adminer:** [http://localhost:8092](http://localhost:8092)
4.  **Stopping Services:**
    ```bash
    docker-compose down
    ```
5.  **View Logs:**
    ```bash
    docker-compose logs -f
    ```

### Cloud Deployment (GKE)

The cloud deployment is fully automated.

1.  **Trigger:** A push to the `main` branch of the connected GitHub repository automatically starts a new build in Google Cloud Build.
2.  **Process:** The `cloudbuild.yaml` file orchestrates the following steps:
    *   Builds the Drupal and Adminer Docker images.
    *   Tags the images with the current git commit SHA and pushes them to Google Artifact Registry.
    *   Authenticates to the GKE cluster.
    *   Replaces placeholder values in the Kubernetes manifests.
    *   Retrieves the database password from Google Secret Manager and creates a Kubernetes secret.
    *   Applies the manifests in the `k8s/` directory to deploy the application.
3.  **Monitoring:** The build and deployment process can be monitored from the **Cloud Build -> History** page in the Google Cloud Console.

## Development Conventions

*   **Infrastructure as Code:** All environment and deployment configurations are version-controlled in this repository. Changes to the cloud environment should be made by modifying these files, not through manual changes in the Cloud Console.
*   **Containerization:** All application components are containerized using the provided `Dockerfile`s. Local development should be done via `docker-compose` to ensure consistency with the deployed environment.
*   **CI/CD:** All deployments to the GKE environment are handled automatically by the Cloud Build pipeline. Manual deployments via `kubectl apply` are discouraged.
*   **Secret Management:** Sensitive data like database passwords are not stored in the repository. They are managed by Google Secret Manager and injected into the environment at deploy time.
*   **Documentation:** The `DEPLOYMENT_GUIDE.md` file serves as the primary source of truth for the setup, architecture, and troubleshooting of the cloud environment. It should be kept up-to-date with any changes.
