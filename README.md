
## CI/CD to EKS with Helm + ArgoCD (GitHub Actions OIDC)

This repo ships a workflow at [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml) that:
1) Builds and pushes a Docker image.
2) Deploys to EKS using Helm (chart in `./chart`).
3) Syncs the release via ArgoCD.
4) Authenticates to AWS via GitHub OIDC (no long-lived AWS keys).

### Helm assets in this repo
- Chart path: [`chart/`](chart/Chart.yaml) with [`values.yaml`](chart/values.yaml), [Deployment](chart/templates/deployment.yaml), [Service](chart/templates/service.yaml), helpers in [`_helpers.tpl`](chart/templates/_helpers.tpl).
- App listens on `8080` ([`server.js`](server.js:6)), Service exposes port `80` → targetPort `8080`.
- Set `HELM_CHART_PATH=./chart` and `HELM_RELEASE` (e.g., `nodejs-web-app`).

### One-time AWS prep
1) Create/choose OIDC IAM role (trust: `token.actions.githubusercontent.com`, audience: `sts.amazonaws.com`), permissions: ECR push/pull, EKS describe/update kubeconfig, and namespace-level deploy perms.
2) Create or confirm ECR repo: `aws ecr create-repository --repository-name nodejs-web-app --region <region>`.
3) Ensure kubeconfig works locally: `aws eks update-kubeconfig --name <cluster> --region <region>` then `kubectl get ns`.
4) Ensure target namespace exists: `kubectl create namespace <namespace>` (idempotent if already exists).

### GitHub secrets to set (repo → Settings → Secrets and variables → Actions)
- `AWS_ROLE_ARN`: OIDC role ARN you created.
- `AWS_REGION`: e.g., `us-east-1`.
- Prefer `ARGOCD_AUTH_TOKEN` (from `argocd account generate-token ...`), else `ARGOCD_USERNAME` and `ARGOCD_PASSWORD`.
- If using a non-ECR registry, add its creds (e.g., `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`) and adjust login step.

### Fill workflow env placeholders in [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml)
- AWS: `AWS_REGION`, `AWS_ROLE_ARN`
- Cluster: `EKS_CLUSTER_NAME`
- Helm: `EKS_NAMESPACE`, `HELM_RELEASE`, `HELM_CHART_PATH` (use `./chart`), `IMAGE_REPO` (e.g., `<account>.dkr.ecr.<region>.amazonaws.com/nodejs-web-app`)
- ArgoCD: `ARGOCD_SERVER`, `ARGOCD_APP_NAME`, `ARGOCD_DEST_NAMESPACE`, `ARGOCD_PROJECT`, and either `ARGOCD_AUTH_TOKEN` or `ARGOCD_USERNAME`/`ARGOCD_PASSWORD`

### What the workflow does (step-by-step)
1) Checkout, Node 18 setup, `npm ci`, `npm test --if-present`.
2) Configure AWS via OIDC: short-lived creds from `AWS_ROLE_ARN` in `AWS_REGION`.
3) Login to registry (ECR by default), build image, push as `<IMAGE_REPO>:<GITHUB_SHA>`.
4) `aws eks update-kubeconfig` to target cluster.
5) `helm upgrade --install` with `image.repository` and `image.tag` overrides pointing to the pushed image; creates namespace if missing.
6) Install ArgoCD CLI; login via token or username/password.
7) `argocd app create --upsert` using the chart path and image overrides; then `argocd app sync` and `argocd app wait` for health.

### Commands and where values come from
- ECR repo (if absent): `aws ecr create-repository --repository-name nodejs-web-app --region <region>`
- Cluster name: `aws eks list-clusters` or AWS console → EKS.
- Namespace create: `kubectl create namespace <namespace>`
- OIDC role trust: AWS IAM → Roles → Create role → Web identity → Provider `token.actions.githubusercontent.com`; add policy for ECR/EKS/Helm namespace operations.
- ArgoCD token: `argocd account generate-token --account <your-account>` (requires ArgoCD access); or use UI to generate.

### How to run
- Push/PR to `master` triggers automatically.
- Manual run: GitHub → Actions → select workflow → “Run workflow”.
- Adjust branches by editing the `on:` block in the workflow file.

### Local quickstart (Node + Docker)


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

 
