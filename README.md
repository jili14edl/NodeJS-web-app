
Create a simple nodeJs application and deploy it onto a docker container.

## CI/CD to EKS with Helm + ArgoCD (GitHub Actions OIDC)

The workflow at [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml) builds, pushes, and deploys to EKS with Helm, then syncs via ArgoCD. It authenticates to AWS with GitHub OIDC.

### Where the Helm chart lives and how it’s used
- Chart path: [`chart/`](chart/Chart.yaml) with [`values.yaml`](chart/values.yaml) and templates for [Deployment](chart/templates/deployment.yaml) and [Service](chart/templates/service.yaml). Helpers are in [`_helpers.tpl`](chart/templates/_helpers.tpl).
- The workflow’s `HELM_CHART_PATH` should be set to `./chart` (or your chosen path if you move it). It runs `helm upgrade --install` with `--set image.repository` and `--set image.tag` to point to the built image.
- Default app container port is `8080` (matches [`server.js`](server.js:6)), and Service exposes port `80` → targetPort `8080`.

### Values you need (and where to get them)
- `AWS_ROLE_ARN`: IAM role that trusts GitHub OIDC; create in AWS IAM → Roles → “Web identity” with provider `token.actions.githubusercontent.com` and allow ECR/EKS permissions.
- `AWS_REGION`: Region of your EKS cluster/ECR (e.g., `us-east-1`).
- `EKS_CLUSTER_NAME`: From `aws eks list-clusters` or AWS console → EKS → cluster name.
- `EKS_NAMESPACE`: Kubernetes namespace to deploy into (create if needed: `kubectl create ns <name>`).
- `HELM_RELEASE`: Helm release name, e.g., `nodejs-web-app`.
- `HELM_CHART_PATH`: Path to your chart in this repo (e.g., `./chart` or `./deploy/chart`).
- `IMAGE_REPO`: Registry repo, e.g., ECR `$(aws sts get-caller-identity --query Account --output text).dkr.ecr.<region>.amazonaws.com/nodejs-web-app`.
- `ARGOCD_SERVER`: Host:port of ArgoCD API, e.g., `argo.example.com:443`.
- `ARGOCD_USERNAME`/`ARGOCD_PASSWORD` or `ARGOCD_AUTH_TOKEN`: From ArgoCD (Settings → Accounts or CLI `argocd account generate-token`).
- `ARGOCD_APP_NAME`: Desired ArgoCD app name, e.g., `nodejs-web-app`.
- `ARGOCD_DEST_NAMESPACE`: Namespace ArgoCD should deploy to (often same as `EKS_NAMESPACE`).
- `ARGOCD_PROJECT`: ArgoCD project (default is `default`).

### One-time AWS/ECR/EKS prep
1) Ensure an ECR repo exists (or create): `aws ecr create-repository --repository-name nodejs-web-app --region <region>`.
2) Ensure kubeconfig works: `aws eks update-kubeconfig --name <cluster> --region <region>` then `kubectl get ns`.
3) Namespace (if missing): `kubectl create namespace <namespace>`.

### Configure GitHub Secrets
In GitHub repo → Settings → Secrets and variables → Actions → New repository secret:
- `AWS_ROLE_ARN`: your OIDC role ARN.
- `AWS_REGION`: your region.
- Prefer `ARGOCD_AUTH_TOKEN`; otherwise set `ARGOCD_USERNAME` and `ARGOCD_PASSWORD`.
- If using a non-ECR registry, add its credentials as needed (e.g., `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`).

### Edit workflow env placeholders
Open [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml) and set:
- `AWS_REGION`, `AWS_ROLE_ARN`, `EKS_CLUSTER_NAME`
- `EKS_NAMESPACE`, `HELM_RELEASE`, `HELM_CHART_PATH`
- `IMAGE_REPO`
- `ARGOCD_SERVER`, `ARGOCD_USERNAME`, `ARGOCD_PASSWORD` **or** use `secrets.ARGOCD_AUTH_TOKEN`
- `ARGOCD_APP_NAME`, `ARGOCD_DEST_NAMESPACE`, `ARGOCD_PROJECT`

### How the workflow runs
1) Checkout, install Node 18, run `npm ci` and `npm test --if-present`.
2) Assume AWS role via OIDC, log in to registry (ECR by default).
3) Build & push image tagged with commit SHA.
4) Update kubeconfig for EKS.
5) `helm upgrade --install` with `image.repository` and `image.tag` overrides.
6) Install ArgoCD CLI, log in, create/update app (upsert), sync, and wait for health.

### Triggers and manual runs
- Default: on `push` and `pull_request` to `master`.
- To run manually: Actions tab → select this workflow → “Run workflow”.
- To target other branches, edit the `on:` block in [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml).

### Local quickstart (unchanged tutorial below)


5. Create a **Dockerfile**
    > touch Dockerfile
6. Dockerfile should look like this
```
FROM node:10
# Create app directory
WORKDIR /usr/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production

# Bundle app source
COPY . .
EXPOSE 3000
CMD [ "node", "server.js" ]

```
7. Create **.dockerignore** file with following content
 
```
node_modules
npm-debug.log
```

```
13. If you followed the  above steps on your system, you will see the sam output as below image: [http://localhost:49160/](http://localhost:49160/)

 
