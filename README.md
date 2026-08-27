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

The workflows are visible in [`.github/workflows`](.github/workflows) and in the repository Actions tab:

- [`frontend-ci.yml`](.github/workflows/frontend-ci.yml): pull requests to `main`, manual runs, parallel lint/test jobs, then a gated build.
- [`backend-ci.yml`](.github/workflows/backend-ci.yml): the equivalent backend checks using Pipenv, pytest, flake8, and a Docker build.
- [`frontend-cd.yml`](.github/workflows/frontend-cd.yml): pushes to `main` and manual runs; reuses frontend CI, pushes an ECR image tagged with the commit SHA, then deploys the Kubernetes manifests to EKS.
- [`backend-cd.yml`](.github/workflows/backend-cd.yml): the equivalent backend image build and EKS deployment.

CI workflows are limited to changes in their own application and workflow file. CD workflows are limited to application changes on `main`. Every CD image uses `${{ github.sha }}`, and deployment substitutes that same immutable tag into the Kubernetes manifest before applying it.

### GitHub configuration

Set these repository or environment **Variables**:

- `AWS_REGION` (optional; defaults to `us-east-1`)
- `ECR_REGISTRY`, for example `123456789012.dkr.ecr.us-east-1.amazonaws.com`
- `EKS_CLUSTER_NAME`
- `MOVIE_API_URL` for the frontend build, such as the deployed backend URL

Set these repository or environment **Secrets**:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

The AWS identity must be allowed to push to both ECR repositories and update the configured EKS cluster. No credential values belong in this repository. OIDC can replace access-key secrets later if the AWS account is configured for it.

To verify Actions, open the repository's **Actions** tab, run either CI workflow with **Run workflow**, and inspect the gated jobs. A pull request changing only the frontend should show frontend CI; a push to `main` changing the backend should show backend CD. CD requires a configured AWS account, ECR registry, EKS cluster, kubeconfig permissions, and the variables/secrets above.

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
