# Movie Picture Pipeline

CI/CD pipelines built with GitHub Actions for the Movie Picture catalog, a web app made up of a React frontend and a Flask backend API. Every change is linted, tested, and built automatically, and merges to `main` are deployed to an AWS EKS (Kubernetes) cluster.

**Repository:** https://github.com/aarcha-2545/cd12355-Movie-Picture-Pipeline

## Live Applications

| App      | URL |
| -------- | --- |
| Frontend | http://a7f6ab208c56d4b6b9abed31ceca4da0-14598355.us-east-1.elb.amazonaws.com |
| Backend  | http://a35f3c9c5d8a8429c901e843040765f6-949209305.us-east-1.elb.amazonaws.com/movies |

> The AWS infrastructure was destroyed after testing to avoid costs, so these URLs may no longer respond. See [Evidence](#evidence) for screenshots of both apps working.

## Architecture

| Component | Stack | Location |
| --------- | ----- | -------- |
| Frontend  | React (TypeScript/JavaScript), ESLint, Jest | `starter/frontend` |
| Backend   | Python, Flask, pytest, flake8 | `starter/backend` |
| Infrastructure | Terraform (ECR, EKS, IAM) | `setup/terraform` |
| Deployment | Docker images in ECR, Kubernetes manifests applied with kustomize | `starter/*/k8s` |

## Workflows

All workflows live in `.github/workflows/` and can also be started manually from the Actions tab (`workflow_dispatch`).

| Workflow | Trigger | Jobs |
| -------- | ------- | ---- |
| Backend CI | Pull request to `main` touching `starter/backend/**` | `lint` and `test` run in parallel, then `build` |
| Backend CD | Push to `main` touching `starter/backend/**` | `lint` and `test` in parallel, then `build` (image tagged with git SHA), then `deploy` |
| Frontend CI | Pull request to `main` touching `starter/frontend/**` | `lint` and `test` run in parallel, then `build` |
| Frontend CD | Push to `main` touching `starter/frontend/**` | `lint` and `test` in parallel, then `build` (image tagged with git SHA), then `deploy` |

### How the pipeline works

1. **Lint** fails the run if the code breaks ESLint (frontend) or flake8 (backend) rules.
2. **Test** fails the run if the Jest (frontend) or pytest (backend) suite fails.
3. **Build** runs only if lint and test both pass (`needs: [lint, test]`). In CD, it builds the Docker image and pushes it to ECR tagged with the commit SHA.
4. **Deploy** runs only after a successful build. It points kustomize at the new image tag and applies the manifests to the cluster:

   ```bash
   kustomize edit set image <app>=<ECR_REPO_URL>:<GIT_SHA>
   kustomize build | kubectl apply -f -
   ```

   The manifest change is made inside the runner and is never committed.

### Required GitHub secrets

| Secret | Purpose |
| ------ | ------- |
| `AWS_ACCESS_KEY_ID` | Access key for the `github-action-user` IAM user |
| `AWS_SECRET_ACCESS_KEY` | Secret key for the same user |
| `AWS_SESSION_TOKEN` | Only if your credentials are temporary |

The frontend image is also built with the backend URL (`REACT_APP_MOVIE_API_URL`) so it knows where to fetch movies.

## Running Locally

### Backend

```bash
cd starter/backend
pipenv install
pipenv run lint
pipenv run test
pipenv run serve          # http://localhost:5000/movies
```

### Frontend

```bash
cd starter/frontend
nvm use
npm ci
npm run lint
CI=true npm test
REACT_APP_MOVIE_API_URL=http://localhost:5000 npm start   # http://localhost:3000
```

### Docker

```bash
# Backend
docker build --tag mp-backend:latest starter/backend
docker run -p 5000:5000 -d mp-backend

# Frontend
docker build --build-arg=REACT_APP_MOVIE_API_URL=http://localhost:5000 \
  --tag mp-frontend:latest starter/frontend
docker run -p 3000:3000 -d mp-frontend
```

## Evidence

Screenshots are in the [`Screenshots/`](Screenshots) folder:

- Backend CI and CD successful runs
- Frontend CI and CD successful runs
- Backend `/movies` endpoint returning the movie list
- Frontend displaying the Movie List
- `kubectl get all` showing healthy deployments, services, and running pods

## Verifying a Deployment

```bash
curl http://<BACKEND_LB_URL>/movies
# {"movies":[{"id":"123","title":"Top Gun: Maverick"},{"id":"456","title":"Sonic the Hedgehog"},{"id":"789","title":"A Quiet Place"}]}
```

Opening the frontend URL in a browser should show the same list under the "Movie List" heading.

## Cleanup

AWS resources cost money while running. When you're done:

```bash
cd setup/terraform
terraform destroy
```

## License

See [LICENSE.md](LICENSE.md).
