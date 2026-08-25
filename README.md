# Go Web App — Complete DevOps Project

## Project Overview

* This project is a complete **DevOps implementation of a Go Web Application**.
* The project covers the complete journey from running the application locally to deploying it on **Amazon EKS** and implementing **CI/CD and GitOps**.
* The overall technologies used in this project are:

  * Go
  * Docker
  * Docker Hub
  * Kubernetes
  * Amazon EKS
  * AWS Load Balancer
  * Ingress
  * Helm
  * GitHub
  * GitHub Actions
  * Argo CD

---

# Project Flow

```text
Go Application
      ↓
Run Locally
      ↓
Dockerize Application
      ↓
Build Docker Image
      ↓
Run Container Locally
      ↓
Push Image to Docker Hub
      ↓
Create Kubernetes Manifests
      ↓
Create Amazon EKS Cluster
      ↓
Deploy Application
      ↓
Service
      ↓
Ingress Controller
      ↓
AWS Load Balancer
      ↓
Custom Domain
      ↓
Helm
      ↓
GitHub Actions
      ↓
Docker Image Build & Push
      ↓
Helm Chart Update
      ↓
Argo CD
      ↓
GitOps Deployment
      ↓
Application Running on EKS
```

---

# 1. Run the Go Application Locally

## Install Go

* Install Go on the Linux machine:

```bash
sudo apt update
sudo apt install golang-go -y
```

## Check Go Version

```bash
go version
```

* This confirms that Go has been installed successfully.

## Go to the Project Directory

```bash
cd go-web-app
```

## Run the Application

```bash
go run .
```

* The Go application should start on port `8080`.

## Test the Application

* Open the browser and access:

```text
http://localhost:8080
```

* If the application opens successfully:

  * Go is installed correctly.
  * The application is running correctly.
  * Port `8080` is working.

---

# 2. Dockerize the Go Application

* Open the `go-web-app` project in **Visual Studio Code**.

* Create a file named:

```text
Dockerfile
```

## Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build the Go application

FROM golang:1.22.5 as base

WORKDIR /app

COPY go.mod .

RUN go mod download 

COPY . .

RUN go build -o main .

# second stage - Distroless image

FROM gcr.io/distroless/base

COPY --from=base /app/main .

COPY --from=base /app/static ./static

EXPOSE 8080

CMD [ "./main" ]
```

---

# 3. Why Use a Multi-Stage Dockerfile?

* The Dockerfile contains two stages.

## Stage 1 — Builder

* Contains the Go compiler.
* Downloads dependencies.
* Copies the source code.
* Builds the Go application.
* Produces the application binary.

## Stage 2 — Runtime

* Does not contain the Go compiler.
* Only the compiled application binary is copied.
* This makes the final image smaller.

## Overall Flow

```text
Go Source Code
      ↓
Go Builder Image
      ↓
go build
      ↓
Go Binary
      ↓
distroless Runtime Image
      ↓
Final Docker Image
```

---

# 4. Build the Docker Image

* Build the Docker image from the project directory:

```bash
docker build -t nav8978/go-web-app:v1 .
```

## Check the Docker Image

```bash
docker images
```

* You should see the image:
---

# 5. Run the Docker Container Locally

* Run the container:

```bash
docker run -p 8080:8080 -it nav8978/go-web-app:v1
```

## Port Mapping

```text
Host
8080
  ↓
Container
8080
```

* Test the application in the browser:

```text
http://localhost:8080/courses
```

* If the application is accessible:

  * Docker image is working.
  * Container is running.
  * Application is listening on port `8080`.

---

# 6. Push the Docker Image to Docker Hub

## Login to Docker Hub

```bash
docker login
```

* Enter the Docker Hub username and password/token.

## Push the Image

```bash
docker push nav8978/go-web-app:v1
```

* The image is now available in Docker Hub.
---

# 7. Create Kubernetes Manifests

* Open the project in Visual Studio Code.

* Create Kubernetes manifest files.

```text
go-web-app/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

## Kubernetes Resources

### Deployment

* Manages the application Pods.
* Ensures the desired number of Pods are running.

### Service

* Provides stable networking to the Pods.
* Routes traffic to the application Pods.

### Ingress

* Provides HTTP/HTTPS routing.
* Uses a hostname to route traffic to the Service.

---

# 8. Create the Amazon EKS Cluster

* Create the EKS cluster using `eksctl`:

```bash
eksctl create cluster --name demo-cluster --region us-east-1
```

## Verify the Cluster

```bash
kubectl get nodes
```

* If the nodes are displayed, `kubectl` is successfully connected to the EKS cluster.

---

# 9. Deploy the Application to EKS

## 1. Create deployment yaml
```yaml
#this is deployment file
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-web-app
  labels:
    app: go-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-web-app
  template:
    metadata:
      labels:
        app: go-web-app
    spec:
      containers:
      - name: go-web-app
        image: nav8978/go-web-app:{{ .Values.image.tag }}
        ports:
        - containerPort: 8080
```
## Now apply the deploymnet
```bash
kubectl apply -f deployment.yaml
```
## 2. Create the service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: go-web-app
  labels:
    app: go-web-app
spec:
  selector:
    app: go-web-app
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP
```
## Now apply the service
```bash
kubectl apply -f service.yaml
```
## 3. create Ingress resource
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-web-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: go-web-app.local
    http:
      paths: 
      - path: /
        pathType: Prefix
        backend:
          service:
            name: go-web-app
            port:
              number: 80
```
## Now apply the ingress 
```bash
kubectl apply -f ingress.yaml
```
---

# 10. Verify the Deployment

## Check Pods

```bash
kubectl get pods
```

## Check Deployment

```bash
kubectl get deployment
```

## Check Service

```bash
kubectl get svc
```

## Check Ingress

```bash
kubectl get ingress
```

* The basic application flow is now:

```text
EKS
 ↓
Deployment
 ↓
Pod
 ↓
Service
 ↓
Ingress
```

---

# 11. Test the Application Using NodePort

* Before testing the Ingress setup, change the Service type to:

```yaml
type: NodePort
```

## Example Service

```yaml
kubectl edit svc <service-name> 
```

## Check the Service

```bash
kubectl get svc
```

* You should see something similar to:

```text
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
go-web-app    NodePort   10.x.x.x        <none>        80:30xxx/TCP
```

* The important part is:

```text
8080:30xxx
```

* `80` → Service port
* `30xxx` → NodePort

---

# 12. Get the EKS Node IP

```bash
kubectl get nodes -o wide
```

* Get the accessible node IP.

* The application can then be tested using:

```text
http://<NODE-IP>:<NODE-PORT>
```

* Example:

```text
http://<NODE-IP>:30080
```

---

# 13. Troubleshoot NodePort Access

* If the application is not accessible, check the following.

## Check Pods

```bash
kubectl get pods -o wide
```

## Check Service

```bash
kubectl get svc go-web-app
```

## Check Endpoints

```bash
kubectl get endpoints go-web-app
```

* If endpoints are available, the Service has Pods behind it.

## Check EndpointSlices

```bash
kubectl get endpointslice
```

---

# 14. Check AWS Security Group

* If NodePort is still not accessible:

  1. Open the AWS Console.
  2. Go to **EC2**.
  3. Open **Security Groups**.
  4. Find the Security Group associated with the EKS nodes.
  5. Check the **Inbound Rules**.
  6. Allow the required NodePort traffic for testing.

* Example:

```text
Protocol: TCP
Port: NodePort
Source: Your IP
```

* Avoid opening unnecessary ports to the entire internet.

---

# 15. Create the Ingress Controller

* The Ingress resource itself does not process traffic.
* An **Ingress Controller** is required.

## Ingress Architecture

```text
Internet
   ↓
AWS Load Balancer
   ↓
Ingress Controller
   ↓
Ingress Resource
   ↓
Service
   ↓
Pod
```

* After installing the Ingress Controller, verify its Pods:

```bash
kubectl get pods -A
```

* Check Services:

```bash
kubectl get svc -A
```

---

# 16. Check the Ingress Resource

```bash
kubectl get ingress
```

* Get detailed information:

```bash
kubectl describe ingress
```

* The Ingress can contain a custom hostname such as:

```text
go-web-app.local
```

## Example

```yaml
spec:
  rules:
    - host: go-web-app.local
```

---

# 17. Why `go-web-app.local` Does Not Work Automatically

* `go-web-app.local` is a custom local hostname.
* It is not automatically registered in public DNS.
* Your computer needs to know which IP address should be used for this hostname.

## Required Mapping

```text
go-web-app.local
        ↓
Ingress Load Balancer IP
```

---

# 18. Find the Load Balancer Address

* Check the Ingress:

```bash
kubectl get ingress
```

* The `ADDRESS` field should show the address associated with the Ingress.

* Use the actual address returned by your AWS environment.

---

# 19. Configure the Windows Hosts File

* If accessing the application from Windows, manually map the custom hostname.

## Open Notepad as Administrator

* Open **Start**.
* Search for **Notepad**.
* Right-click **Notepad**.
* Select **Run as administrator**.

## Open the Hosts File

* Select:

```text
File → Open
```

* Navigate to:

```text
C:\Windows\System32\drivers\etc\
```
here add the custom domain entry with ip which we get using nslookup.

---

# 20. Add the Domain Mapping

* Add:

```text
<LOAD-BALANCER-IP> go-web-app.local
```

* Example:

```text
98.89.53.104 go-web-app.local
```

* The IP above is only an example.
* Always use the actual Load Balancer address from your environment.

---

# 21. Flush Windows DNS

* Open **Command Prompt as Administrator**.

* Run:

```cmd
ipconfig /flushdns
```

* This clears the local DNS cache.

---

# 22. Access the Application Using the Custom Domain

* Open the browser:

```text
http://go-web-app.local
```

## Complete Request Flow

```text
Browser
   ↓
go-web-app.local
   ↓
Windows hosts file
   ↓
AWS Load Balancer
   ↓
Ingress Controller
   ↓
Ingress
   ↓
Service
   ↓
Pod
   ↓
Go Application
```

---

# 23. Convert the Kubernetes Application to Helm

* Once the application is working on Kubernetes, package the Kubernetes resources using Helm.

## Create the Helm Chart

```bash
helm create go-web-app-chart
```

* This creates:

```text
go-web-app-chart/
├── Chart.yaml
├── values.yaml
├── charts/
├── templates/
└── .helmignore
```

---

# 24. Add the Helm Chart to the Go Web App Project


# 25. Move Kubernetes Resources into Helm

* Remove the default files inside:

```text
go-web-app-chart/templates/
```

* Copy the application's Kubernetes resource definitions into the Helm `templates` directory.

* The Helm chart can contain:

```text
templates/
├── deployment.yaml
├── service.yaml
└── ingress.yaml
```

* These are now Helm templates rather than standalone Kubernetes manifests.

---

# 26. Configure `values.yaml`

* Application-specific values should be maintained in:

```text
values.yaml
```

* Example:

```yaml
# Default values for go-web-app-chart.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nav8978/go-web-app
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "32869882699"

ingress:
  enabled: false
  className: ""
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

* The templates use these values instead of hard-coding everything.

---

# 27. Remove the Old Standalone Kubernetes Resources

* After moving the application to Helm, Helm should manage the Kubernetes resources.

* Do not continue maintaining duplicate resources through both:

* The goal is:

```text
Helm Chart
     ↓
Kubernetes Resources
```

---

# 28. Uninstall the Helm Release

* If a Helm release already exists:

```bash
helm uninstall go-web-app
```

* Verify:

```bash
helm list
```

---

# 29. Validate the Helm Chart


## Install the Chart

```bash
helm install go-web-app ./go-web-app-chart
```

## Verify

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

---

# 30. GitHub Actions CI/CD

* The next stage is to automate the build and deployment process using **GitHub Actions**.

## Create Workflow Directory

```text
.github/
└── workflows/
```

## Create Workflow File

```text
.github/workflows/ci.yaml
```

* The workflow will automate tasks such as:

```text
Developer Push
      ↓
GitHub
      ↓
GitHub Actions
      ↓
Build Docker Image
      ↓
Push Image to Docker Hub
      ↓
Update Helm Chart
      ↓
Commit Changes
```

---

# 31. Create GitHub Actions Secrets

* Go to the GitHub repository.

* Navigate to:

```text
Settings
   ↓
Secrets and variables
   ↓
Actions
```

* Add the required secrets.

## Docker Hub Credentials

```text
DOCKER_USERNAME
DOCKER_PASSWORD
```

* `DOCKER_PASSWORD` should preferably contain a **Docker Hub access token** rather than the normal account password.

---

# 32. Create Docker Hub Access Token

* Go to Docker Hub.
* Open account settings.
* Create an access token.
* Copy the token.
* Add it to GitHub Actions secrets.

```text
DOCKER_PASSWORD = <Docker Hub Access Token>
```

* Never put the token directly inside the GitHub Actions YAML file.

---

# 33. Create GitHub Personal Access Token

* If the GitHub Actions workflow needs to update the Helm chart and push changes back to GitHub:

  * Create a GitHub Personal Access Token.
  * Give it only the required permissions.
  * Store it as a GitHub Actions secret.

* Example:

```text
GH_TOKEN
```

* Never commit the token into the repository.

---

# 34. Check Git Remote

* Before pushing changes, always verify the remote repository:

```bash
git remote -v
```

* Make sure the correct GitHub repository is configured.

---

# 35. Pull Latest Changes

* Check the current status:

```bash
git status
```

* Pull remote changes before pushing:

```bash
git pull --rebase origin main
```

* If conflicts occur, resolve them before continuing.

---

# 36. Commit the Project

## Add Files

```bash
git add .
```

## Check Status

```bash
git status
```

## Commit

```bash
git commit -m "feat: implement CI/CD"
```

---

# 37. Push to GitHub

```bash
git push origin main
```

* Go to the GitHub repository.

* Open:

```text
Actions
```

* Verify that the workflow has started.

---

# 38. Install Argo CD

* Argo CD will be used for **GitOps-based continuous delivery**.

## Create Argo CD Namespace

```bash
kubectl create namespace argocd
```

## Install Argo CD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

# 39. Verify Argo CD

```bash
kubectl get pods -n argocd
```

* Wait until the Argo CD components are running.

* Check all resources:

```bash
kubectl get all -n argocd
```

---

# 40. Expose Argo CD Using a Load Balancer

* Change the `argocd-server` Service to `LoadBalancer`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

* Check the Service:

```bash
kubectl get svc argocd-server -n argocd
```

* AWS will provision a Load Balancer.

---

# 41. Final GitOps Flow

```text
Developer
    ↓
Git Push
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Build Docker Image
    ↓
Push Image to Docker Hub
    ↓
Update Helm Chart
    ↓
GitHub Repository
    ↓
Argo CD
    ↓
Detect Git Changes
    ↓
Sync Application
    ↓
Amazon EKS
    ↓
Kubernetes
    ↓
Go Web Application
```

---

# 42. Complete Project Architecture

```text
                         ┌─────────────────┐
                         │    Developer    │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │     GitHub      │
                         └────────┬────────┘
                                  │
                                  ▼
                       ┌─────────────────────┐
                       │   GitHub Actions    │
                       │        CI/CD        │
                       └──────────┬──────────┘
                                  │
                     ┌────────────┴────────────┐
                     ▼                         ▼
             ┌───────────────┐          ┌──────────────┐
             │   Docker Hub  │          │ Helm Chart   │
             │ Docker Image  │          │ Repository   │
             └───────┬───────┘          └──────┬───────┘
                     │                         │
                     │                         │
                     └────────────┬────────────┘
                                  ▼
                           ┌──────────────┐
                           │   Argo CD    │
                           │   GitOps     │
                           └──────┬───────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │   Amazon EKS    │
                         └────────┬────────┘
                                  │
                       ┌──────────┴──────────┐
                       ▼                     ▼
                ┌──────────────┐      ┌──────────────┐
                │  Deployment  │      │    Ingress   │
                └──────┬───────┘      └──────┬───────┘
                       │                     │
                       ▼                     ▼
                  ┌─────────┐         ┌──────────────┐
                  │   Pods  │         │Load Balancer │
                  └────┬────┘         └──────┬───────┘
                       │                     │
                       ▼                     │
                  ┌─────────┐                │
                  │ Service │◄───────────────┘
                  └────┬────┘
                       │
                       ▼
                ┌──────────────┐
                │ Go Web App   │
                └──────────────┘
```

---

# 43. Technology Used

| Technology            | Purpose                        |
| --------------------- | ------------------------------ |
| **Go**                | Application development        |
| **Docker**            | Containerization               |
| **Docker Hub**        | Container image registry       |
| **Kubernetes**        | Container orchestration        |
| **Amazon EKS**        | Managed Kubernetes cluster     |
| **Service**           | Exposes Pods inside Kubernetes |
| **NodePort**          | Temporary external testing     |
| **Ingress**           | HTTP/HTTPS traffic routing     |
| **AWS Load Balancer** | External traffic entry point   |
| **Helm**              | Kubernetes package management  |
| **GitHub**            | Source code management         |
| **GitHub Actions**    | CI/CD automation               |
| **Argo CD**           | GitOps continuous delivery     |

---

# 44. Important Commands

## Go

```bash
sudo apt update
sudo apt install golang-go -y
go version
go run .
```

## Docker

```bash
docker build -t nav8978/go-web-app:v1 .
docker run -p 8080:8080 -it nav8978/go-web-app:v1
docker login
docker push nav8978/go-web-app:v1
```

## EKS

```bash
eksctl create cluster --name demo-cluster --region us-east-1
kubectl get nodes
```

## Kubernetes

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

kubectl get pods
kubectl get deployment
kubectl get svc
kubectl get ingress
```

## Helm

```bash
helm create go-web-app-chart
helm lint ./go-web-app-chart
helm template go-web-app ./go-web-app-chart
helm install go-web-app ./go-web-app-chart
helm list
helm uninstall go-web-app
```

## Git

```bash
git remote -v
git status
git pull --rebase origin main
git add .
git commit -m "feat: implement CI/CD"
git push origin main
```

## Argo CD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc argocd-server -n argocd
```

---

# 45. Final Project Understanding

* The project starts with a **Go application** running locally.
* Docker is then used to package the application.
* The Docker image is pushed to **Docker Hub**.
* Kubernetes manifests are created for the application.
* The application is deployed to **Amazon EKS**.
* A Service provides networking to the Pods.
* NodePort can be used to initially test external connectivity.
* An Ingress Controller and Load Balancer provide proper external HTTP access.
* A custom hostname such as `go-web-app.local` can be mapped locally using the Windows `hosts` file.
* Helm is then introduced to package and manage the Kubernetes resources.
* GitHub Actions automates the CI/CD process.
* Docker Hub stores the newly built container images.
* The Helm chart is maintained in Git.
* Argo CD continuously monitors Git and synchronizes the desired state into EKS.
* This results in a complete **CI/CD + GitOps DevOps workflow**.

---

# End-to-End DevOps Pipeline

```text
                    GO WEB APPLICATION
                            │
                            ▼
                    ┌──────────────┐
                    │     Go       │
                    │  Application │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Docker    │
                    │ Multi-Stage  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Docker Hub  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Amazon EKS │
                    └──────┬───────┘
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
          ┌─────────────┐     ┌─────────────┐
          │ Deployment  │     │   Ingress   │
          └──────┬──────┘     └──────┬──────┘
                 │                   │
                 ▼                   ▼
              ┌──────┐         ┌────────────┐
              │ Pods │         │LoadBalancer│
              └──┬───┘         └─────┬──────┘
                 │                   │
                 └────────┬──────────┘
                          ▼
                    Go Web Application
                          │
                          ▼
                    Custom Domain
                          │
                          ▼
                         Helm
                          │
                          ▼
                    GitHub Actions
                          │
                          ▼
                       Argo CD
                          │
                          ▼
                     GitOps on EKS
```
