## Project Description

DEVOPS-GITOPS-ArgoCD-CICD is a complete end-to-end DevOps demo that takes a simple Java Servlet (packaged as a WAR), containerizes it with Docker, and deploys it to Kubernetes using GitOps with Argo CD.

#### The project includes:

- A Java web application (pom.xml + servlet code + JSP) built with Maven into target/java-example.war.
- A Docker workflow (dockerfile) that runs the WAR on Apache Tomcat.
- A Helm chart (argocdhelm/) that defines the Kubernetes Deployment and Service for the application.
- A GitHub Actions CI/CD pipeline (.github/workflows/main.yml) that:
  1. Builds the app with Maven,
  2. Builds and pushes the Docker image to Docker Hub,
  3. Updates argocdhelm/values.yaml with the new image tag,
  4. Lets Argo CD detect and deploy the updated Helm values automatically (GitOps approach).
  
The goal is to demonstrate how CI builds an immutable image, Git tracks the deployment configuration change (Helm values), and Argo CD performs the Kubernetes rollout based on the repository state.


## Overview
This repo contains a small Java Servlet webapp (packaged as a WAR) plus:

1. A `Dockerfile` to build/run it with Tomcat
2. A Helm chart in `argocdhelm/` to deploy the Docker image to Kubernetes
3. A GitHub Actions workflow in `.github/workflows/main.yml` that:
   - builds & pushes the Docker image to Docker Hub
   - updates `argocdhelm/values.yaml` (the image tag)
   - relies on Argo CD to sync the Helm chart

## What the app does
- `GET /` serves the bundled `index.jsp`
- `GET /hello` returns `Hello, World!` via `HelloWorldServlet`

Tomcat in the provided `Dockerfile` listens on port `8080`.

## Local execution (Docker)
### 1) Build the WAR with Maven
```bash
mvn clean package
```
The WAR is created as:
- `target/java-example.war` (this is what the Dockerfile copies)

### 2) Build the Docker image
```bash
docker build -t kubeserve:local .
```

### 3) Run the container
```bash
docker run -d -p 8080:8080 --name kubeserve-local kubeserve:local
```

### 4) Test
- `http://localhost:8080/`
- `http://localhost:8080/hello`

## Kubernetes deployment with Helm
### Helm chart location
- Chart directory: `argocdhelm/`
- Templates:
  - `argocdhelm/templates/deployment.yml`
  - `argocdhelm/templates/service.yml`

### Chart values
```yaml
# argocdhelm/values.yaml
image:
  repository: leaddevops/kubeserve
  tag: 1
```

### Install / upgrade (example)
```bash
helm upgrade --install kubeserve ./argocdhelm \
  --set image.tag=1
```

### Important port note (read before you deploy)
The provided `Dockerfile` exposes/uses Tomcat port `8080`.
However, the current Helm `service.yml` targets port `80`:
- `argocdhelm/templates/service.yml`: `port: 80`, `targetPort: 80`

If your built image runs on `8080` (as in this repo), update the Service to use `targetPort: 8080`.
If you are deploying a different image that listens on `80`, then you can keep it as-is.

## CI/CD with GitHub Actions + Argo CD
### Workflow file
- `.github/workflows/main.yml`
- Workflow name: `CICD with ArgoCD`
- Trigger: `push` to `main` and `master`

### What the workflow does
1. Builds the Java code with `mvn package`
2. Builds and pushes a Docker image to Docker Hub tagged with `github.run_number`
3. Updates `argocdhelm/values.yaml`:
   - it changes `image.tag` to `${{ github.run_number }}`
   - commits & pushes the change back to the repo
4. Argo CD should automatically deploy the updated Helm chart (if Argo CD is set up to track this repo/path)

### Required GitHub secrets
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN` (Docker Hub access token / password)

### Align Docker image name vs Helm image repository
The workflow builds/pushes using:
- `${{ secrets.DOCKERHUB_USERNAME }}/${{ env.imageName }}:${{ github.run_number }}`

But the Helm chart uses:
- `argocdhelm/values.yaml -> image.repository: leaddevops/kubeserve`

To make CI/CD + Helm match, ensure both point to the same Docker Hub repository, for example:
- set `env.imageName: kubeserve`
- set `DOCKERHUB_USERNAME` to `leaddevops`
Or update `argocdhelm/values.yaml` to match your chosen Docker Hub repo.

### Argo CD setup (conceptual)
In Argo CD, create an Application that points to:
- `repoURL`: your GitHub repo URL
- `path`: `argocdhelm` (the Helm chart directory)
- `helm`: use the chart’s `values.yaml` (Argo CD will pick up tag updates committed by GitHub Actions)

Enable automated sync if you want deployments to happen automatically after each push.

### Recommended safeguard: avoid workflow loops
This workflow commits back to the same branch (`git push`), and the workflow triggers on `push`.
That can cause repeated runs.
A common fix is to ignore commits that only change `argocdhelm/values.yaml`, or add a conditional guard.

## Files worth knowing
- `pom.xml`: Maven WAR build config (WAR name: `java-example.war`)
- `dockerfile`: builds Tomcat image and runs the WAR as `ROOT.war`
- `argocdhelm/`: Helm chart used by Argo CD
- `.github/workflows/main.yml`: CI/CD pipeline that updates the Helm tag
