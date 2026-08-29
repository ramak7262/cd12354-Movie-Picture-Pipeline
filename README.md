# Movie Picture Pipeline

Movie Picture Pipeline is a small movie catalog application used to demonstrate automated testing, container builds, and Kubernetes delivery with GitHub Actions.

Public repository: https://github.com/ramak7262/cd12354-Movie-Picture-Pipeline

## Architecture

```text
React frontend (port 3000) -> Flask API (port 5000) -> in-memory movie data
                                      |
                         Docker images in Amazon ECR
                                      |
                              Amazon EKS / Kubernetes
```

The application lives under `starter/`:

- `starter/frontend`: React 18 application, npm, ESLint, and Jest.
- `starter/backend`: Flask application, Pipenv, pytest, and flake8.
- `starter/*/Dockerfile`: production container definitions.
- `starter/*/k8s`: Kubernetes Service and Deployment manifests.
- `setup/terraform`: optional AWS infrastructure for ECR and EKS.

## Run Locally

### Frontend

```bash
cd starter/frontend
nvm use                 # Node version is pinned in .nvmrc
npm ci
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start
```

### Backend

Python 3.10 and Pipenv are expected by `Pipfile` and `Pipfile.lock`:

```bash
cd starter/backend
pipenv install --dev --deploy
pipenv run serve
```

The API is available at `http://localhost:5000/movies` and the frontend at `http://localhost:3000`.

## Tests and Quality Checks

```bash
cd starter/frontend
npm ci
npm run lint
CI=true npm test -- --watchAll=false
npm run build

cd ../backend
pipenv install --dev --deploy
pipenv run lint
pipenv run test
```

The backend test and lint jobs use Python 3.10 in GitHub Actions, matching the locked project environment.

## GitHub Actions CI/CD

The workflows are visible in [`.github/workflows`](.github/workflows) and in the repository **Actions** tab. There are exactly **four** workflows:

### CI Workflows (Pull Requests)

**`frontend-ci.yml`**: Runs on pull requests to `main` (path: `starter/frontend/**`), manual trigger via **Run workflow**, or workflow file changes.

Jobs (lint and test run in parallel):
1. **lint** — `npm ci` → `npm run lint` (ESLint)
2. **test** — `npm ci` → `CI=true npm test -- --watchAll=false` (Jest)
3. **build** (depends: lint, test) — `npm run build` (production bundle)
4. **docker-build** (depends: lint, test) — Builds `starter/frontend/Dockerfile` without pushing to ECR

Node version: pinned from `.nvmrc` (18.14)

**`backend-ci.yml`**: Equivalent backend checks triggered on pull requests to `main` (path: `starter/backend/**`), manual trigger, or workflow changes.

Jobs (lint and test run in parallel):
1. **lint** — `pipenv install --dev --deploy` → `pipenv run lint` (flake8)
2. **test** — `pipenv install --dev --deploy` → `pipenv run test` (pytest)
3. **build** (depends: lint, test) — Builds `starter/backend/Dockerfile` without pushing

Python version: 3.10 (locked in Pipfile)

### CD Workflows (Continuous Deployment to EKS)

**`frontend-cd.yml`**: Runs on pushes to `main` when `starter/frontend/**` changes, or manual **Run workflow**.

Jobs:
1. **lint** — ESLint (`npm run lint`)
2. **test** — Jest (`npm test`)
3. **build** (depends: lint, test) — `npm run build`
4. **publish** (depends: build) — Configures AWS credentials, logs in to ECR, builds and pushes Docker image tagged with `${{ github.sha }}`, passes `REACT_APP_MOVIE_API_URL` build argument
5. **deploy** (depends: publish) — Updates kubeconfig, applies Kubernetes manifests, replaces image placeholder with ECR image using github.sha
6. **verify-deployment** (depends: deploy) — Polls `kubectl rollout status deployment/frontend --timeout=180s` (allows non-zero exit via `|| true`), logs `kubectl get all` and `kubectl describe deploy frontend`
7. **verify-image** (depends: deploy) — Queries ECR: `aws ecr describe-images --repository-name frontend`, displays tags, digest, and push timestamp

**`backend-cd.yml`**: Equivalent CD workflow triggered on pushes to `main` when `starter/backend/**` changes, or manual **Run workflow**.

Jobs:
1. **lint** — flake8 (`pipenv run lint`)
2. **test** — pytest (`pipenv run test`)
3. **build** (depends: lint, test) — Docker build
4. **publish** (depends: build) — Configures AWS credentials, logs in to ECR, builds and pushes Docker image tagged with `${{ github.sha }}`
5. **deploy** (depends: publish) — Updates kubeconfig, applies Kubernetes manifests, replaces image placeholder with ECR image
6. **verify-deployment** (depends: deploy) — Polls `kubectl rollout status deployment/backend --timeout=180s`, logs `kubectl get all` and `kubectl describe deploy backend`
7. **verify-image** (depends: deploy) — Queries ECR: `aws ecr describe-images --repository-name backend`, displays image details

**Image Tags**: All CD workflows use `${{ github.sha }}` as the Docker image tag. This ensures each deployment uses an immutable, auditable image corresponding to the exact commit.

**Kubernetes Deployment**: CD workflows apply manifests from `starter/frontend/k8s` and `starter/backend/k8s`. Before applying, they replace the placeholder image name (e.g., `frontend` → `<ECR_REGISTRY>/frontend:<github.sha>`) using `kubectl kustomize` and `sed`.

### GitHub Repository Configuration

Before triggering any CD workflow, set these **Variables** in the GitHub repository (Settings → Secrets and variables → Actions → Variables):

- `AWS_REGION`: AWS region where ECR and EKS are provisioned (default: `us-east-1`; optional)
- `ECR_REGISTRY`: ECR login URI, e.g., `123456789012.dkr.ecr.us-east-1.amazonaws.com`
- `EKS_CLUSTER_NAME`: Name of the EKS cluster (e.g., `udacity-eks`)
- `MOVIE_API_URL`: Backend API URL accessible to the frontend at build time, e.g., `http://<backend-load-balancer-ip>/` or `https://backend.example.com`

Set these **Secrets** in the GitHub repository (Settings → Secrets and variables → Actions → Secrets):

- `AWS_ACCESS_KEY_ID`: IAM access key for the GitHub Actions user
- `AWS_SECRET_ACCESS_KEY`: IAM secret access key

**Important**: Do not commit credential values to the repository. AWS credentials and sensitive configuration must exist only in GitHub Secrets/Variables.

The AWS IAM identity must have permissions to:
- Push images to ECR repositories (`frontend`, `backend`)
- Update EKS cluster (create/update deployments, services)
- Read ECR repositories for verification

For local EKS access, run `setup/init.sh` after Terraform provisioning to grant the IAM user kubeconfig access.

### Deployment verification

After the CD workflows publish a Docker image and deploy to EKS, the final verification jobs log the rollout status and ECR image details:

- **Verify Kubernetes deployment**: Runs `kubectl rollout status` with a 180-second timeout to confirm the deployment is ready, then logs `kubectl get all` and `kubectl describe deploy <deployment-name>` for debugging. Uses `|| true` to allow non-zero exit if cluster is unavailable, but real errors are still logged.
- **Verify ECR image**: Queries ECR to display the latest pushed image's tags, digest, and push timestamp (`imagePushedAt`), confirming the image was published successfully.

Verification output is visible in the GitHub Actions workflow logs and provides evidence for Udacity review.

### Testing and Validation

**CI Workflows** (no AWS required):
- Test frontend CI: Make a pull request changing `starter/frontend/**` files
- Test backend CI: Make a pull request changing `starter/backend/**` files
- Or use GitHub UI **Actions** → **Run workflow** to manually trigger

**CD Workflows** (requires AWS/EKS configured):
- Push to `main` changing `starter/frontend/**` triggers frontend CD
- Push to `main` changing `starter/backend/**` triggers backend CD
- Or use GitHub UI **Actions** → **Run workflow** to manually trigger
- Observe workflow logs in **Actions** tab
- After successful deployment, use `kubectl get svc` to find LoadBalancer endpoints

**Backend URL**:
```bash
# After EKS deployment, get backend service IP
kubectl get svc backend
# API endpoint
BACKEND_IP=$(kubectl get svc backend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$BACKEND_IP/movies
```

**Frontend URL**:
```bash
# After EKS deployment, get frontend service IP  
kubectl get svc frontend
# Browser: http://<frontend-load-balancer-hostname>
```

## Deployment Infrastructure

The Terraform in `setup/terraform` provisions the course environment, including ECR repositories and an EKS cluster. Terraform `1.3.9` is required by `versions.tf`. Review its variables and outputs before applying it:

```bash
cd setup/terraform
terraform init
terraform fmt -check
terraform validate
terraform apply
terraform output
```

Use `setup/init.sh` to grant the GitHub Actions IAM user Kubernetes access when using the provided IAM setup. Destroy temporary AWS infrastructure when it is no longer needed to avoid charges. Deployment has not been executed from this workspace because AWS credentials and a live cluster are not available here.

## Limitations and Assumptions

- Movie data is held in memory and is reset when the backend restarts.
- The CD workflows assume the ECR repositories and EKS cluster already exist.
- Kubernetes services are `LoadBalancer` services; set `MOVIE_API_URL` to the backend load balancer URL before publishing the frontend image.
- The pinned legacy frontend toolchain reports upstream npm audit and Browserslist maintenance warnings; changing those dependencies would be a separate upgrade.

## License

See [LICENSE.md](LICENSE.md).
