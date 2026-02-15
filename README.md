# Dockerizing-a-NodeJS-web-app
<p align="center">
    <img
      alt="Node.js"
      src="nodeJsDocker.png"
      width="400"
    />
</p>
<hr>

Create a simple nodeJs application and deploy it onto a docker container.

## CI/CD to EKS with Helm + ArgoCD (GitHub Actions OIDC)

The repository includes a GitHub Actions workflow at [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml) that builds, pushes, and deploys the app to an EKS cluster using Helm, then syncs the release via ArgoCD. It assumes AWS authentication via GitHub OIDC.

### Prerequisites
- An AWS IAM role trusted for GitHub OIDC with permissions for ECR (or your registry), EKS, and Helm-managed namespaces.
- An EKS cluster reachable with `aws eks update-kubeconfig`.
- A Helm chart for the app (path inside this repo, e.g., `./chart`).
- ArgoCD accessible (username/password or auth token).

### Configure the workflow
Edit the env placeholders in [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml):
- `AWS_REGION`, `AWS_ROLE_ARN`, `EKS_CLUSTER_NAME`
- `EKS_NAMESPACE`, `HELM_RELEASE`, `HELM_CHART_PATH`
- `IMAGE_REPO` (e.g., `<aws_account_id>.dkr.ecr.<region>.amazonaws.com/nodejs-web-app`)
- `ARGOCD_SERVER`, `ARGOCD_USERNAME`, `ARGOCD_PASSWORD` **or** `secrets.ARGOCD_AUTH_TOKEN`
- `ARGOCD_APP_NAME`, `ARGOCD_DEST_NAMESPACE`, `ARGOCD_PROJECT`

### Set GitHub secrets (recommended)
- `AWS_ROLE_ARN`, `AWS_REGION`
- `ARGOCD_AUTH_TOKEN` (preferred) or `ARGOCD_USERNAME` and `ARGOCD_PASSWORD`
- If using a private registry different from ECR, add its credentials as needed.

### What the workflow does
1) Checkout, install Node 18, run `npm ci` and `npm test --if-present`.
2) Assume AWS role via OIDC, log in to registry (ECR by default).
3) Build and push image tagged with `GITHUB_SHA`.
4) Update kubeconfig for the target EKS cluster.
5) `helm upgrade --install` with `image.repository` and `image.tag` overrides.
6) Install ArgoCD CLI, log in, create/update the ArgoCD app (upsert), then `argocd app sync` and wait for health.

### Triggering
- Automatically runs on `push` and `pull_request` to `master`.
- For other branches or manual runs, adjust the `on:` block in [.github/workflows/eks-helm-argocd-oidc.yml](.github/workflows/eks-helm-argocd-oidc.yml).

### Local quickstart (unchanged tutorial below)

1. Create a working directory
    > mkdir <working_directory_name>
  
2. Running this command in working directory will initialize your project
    > npm init
  
This will create a **package.json** file in the folder, that file contains app dependency packages.

Replace the following code of package.json

```js
  // package.json

  {
    "name": "docker_web_app",
    "version": "1.0.0",
    "description": "Node.js deploy on Docker container",
    "author": "cmuth001@odu.edu",
    "main": "server.js",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.16.1"
    }
  }
```

2. Running this command will install all the dependencies from package.json 
    > npm install
3. Lets create a **server.js** file that defines a web-app using an Express framework.

  
 ```js
    // server.js
    'use strict';
    var express = require('express');
    var app = express();
    app.get('/', function (req, res) {
      res.send('Hello World!');
    });
    app.listen(3000, function () {
      console.log('Example app listening on port 3000!');
    });

 ```
 
4. Lets test the application, run the below command
    > node server.js
  
  If you followed the  above steps on your system, you will see the same output as below image: [http://localhost:3000/](http://localhost:3000/)
  
  <p align="center">
    <img
      alt="Node.js"
      src="image1.png"
      width="400"
    />
  </p>
      
  Now node.js app is running successfully.
   
Lets try running the same node.js application running on the docker container. To run the application on the docker conatiner we need a docker image. 

First, we will create a docker image for the application.
   
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
   This will prevent from copying onto docker image.
    
8. Building Docker image
    > docker build -t node-web-app .
    
    <p align="center">
        <img
          alt="Node.js"
          src="image2.png"
          width="400"
        />
      </p>
9. Check the Docker images
    > docker images
    
    <p align="center">
        <img
          alt="Node.js"
          src="image3.png"
          width="400"
        />
      </p>
10. Run the docker image
    > docker run -p 49160:3000 -d node-web-app
 
11. Get the container id
    > docker ps
    
    <p align="center">
        <img
          alt="Node.js"
          src="image4.png"
          width="400"
        />
      </p>
      
12. Lets know where it is running on
    > docker logs <container_id>
    
```
output: 
    Example app listening on port 3000!
```
13. If you followed the  above steps on your system, you will see the sam output as below image: [http://localhost:49160/](http://localhost:49160/)
  
  <p align="center">
        <img
          alt="Node.js"
          src="image5.png"
          width="400"
        />
      </p>
    
I hope this tutorial helped you get up and running a simple Node.js application on Docker container.
    
 
