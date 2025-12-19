# Repository Guidelines

## Project Structure & Module Organization
- Root Dockerfiles (`Dockerfile.drupal`, `Dockerfile.mariadb`, `Dockerfile.adminer`) define images for the Drupal app, MariaDB, and Adminer. `docker-compose.yml` wires them together for local work.
- Kubernetes manifests live in `k8s/` (`deployment.yml`, `service.yml`, `persistent-volume-claim.yml`) with image and Cloud SQL placeholders replaced during CI/CD.
- `cloudbuild.yaml` builds images, pushes to Artifact Registry, injects secrets, and applies manifests; see `DEPLOYMENT_GUIDE.md` for the full pipeline flow.
- Use the repo root for scripts and config; Drupal site files persist in the container volume (`/var/www/html/sites/default/files`) bound to the PVC.

## Build, Test, and Development Commands
- Local stack: `docker-compose up --build -d` to start Drupal + MariaDB + Adminer; `docker-compose logs -f drupal-award3` to tail app logs; `docker-compose down -v` to stop and clear volumes.
- Image tweaks: edit the relevant Dockerfile and rerun `docker-compose up --build -d` to rebuild.
- CI/CD: `gcloud builds submit --config cloudbuild.yaml --substitutions _REGION=us-central1,_AR_REPO_NAME=drupal-repo,_GKE_CLUSTER_NAME=drupal-cluster,_SQL_INSTANCE_NAME=drupal-db-instance` (align values with your project). Cloud Build handles placeholder substitution and deploy.
- K8s spot checks: after a build, `kubectl get pods` and `kubectl rollout status deploy/drupal` to confirm the new version is live.

## Coding Style & Naming Conventions
- YAML: two-space indentation, lowercase keys, keep placeholders in `__LIKE_THIS__` format until CI fills them.
- Dockerfiles: favor official base images, group related RUN steps, and leave commented package installs if optional.
- Compose/K8s names: keep service/deployment labels in the `drupal/adminer/mariadb` pattern to match selectors and logs.

## Testing Guidelines
- No automated test suite is present; validate changes by running the compose stack and confirming Drupal loads at `http://localhost:8095` and Adminer at `http://localhost:8092`.
- For cluster changes, run `kubectl apply --dry-run=client -f k8s/` before committing to catch YAML issues; confirm pods become `Ready` and Drupal serves content.
- If you add tests, colocate them beside the component or script they cover and name with a `.test` suffix for easy discovery.

## Commit & Pull Request Guidelines
- Commits: concise, imperative subject lines (e.g., `Add MariaDB init hardening`; `Refine Cloud Build substitutions`) with brief bodies when context is non-obvious.
- PRs: include the intent, key changes, deployment impact (DB/schema changes, secret updates, port changes), and screenshots of UI-facing changes when applicable. Link related issues and note any manual steps required post-merge.

## Security & Configuration Tips
- Never commit secrets; database passwords come from Secret Manager during Cloud Build (`drupal-secrets`), and local runs should use `.env` or compose overrides not checked into Git.
- Keep Cloud SQL instance names and Artifact Registry paths consistent with the placeholders in `k8s/deployment.yml` to avoid mismatched rollouts.
- Rotate credentials after sharing demo environments and prune unused images from the registry to reduce surface area.
