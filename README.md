# 🚀 Brain Tasks Application – CI/CD Deployment on AWS EKS

---

## 📌 Project Overview

This project demonstrates the deployment of a production-ready React application using Docker and Kubernetes on Amazon EKS. The application is containerized with Docker, stored in Docker Hub, and automatically built and deployed through AWS CodePipeline and AWS CodeBuild. Monitoring is implemented using Amazon CloudWatch Logs and Container Insights.

The objective is to automate the complete CI/CD pipeline from source code to deployment on a Kubernetes cluster.

---

## Project Architecture

```

GitHub Repository
        │
        ▼
 AWS CodePipeline
        │
        ▼
 AWS CodeBuild
        │
        ├── Build Docker Image
        ├── Push Image to Docker Hub
        └── Deploy to Amazon EKS
                │
                ▼
        Kubernetes Deployment
                │
                ▼
        Kubernetes Service (LoadBalancer)
                │
                ▼
         React Application

```

---

## Prerequisites

Ensure the following tools are installed and configured before deploying the application:

- Git
- Docker
- Docker Hub account
- AWS CLI
- kubectl
- eksctl
- AWS Account with required IAM permissions

---

## Technologies Used

- React (Vite)
- Docker
- Docker Hub
- Kubernetes
- Amazon EKS
- AWS CodeBuild
- AWS CodePipeline
- AWS CloudWatch
- Git & GitHub

---

## IAM Roles & Permissions
This project uses IAM roles to securely enable communication between AWS services without hardcoding credentials.
Roles Used
 ## CodePipeline Service Role

Orchestrates pipeline execution
Fetches source from GitHub
Triggers CodeBuild
Example:
Role_name: AWSCodePipelineServiceRole-us-east-1-eks-deploy-pipeline
### codepipeline-eks-deploy-pipeline-access-to-codebuild-deploy-policy
create policy- give access to Build (StartBuildBatch,StartBuild,BatchGetBuilds) to specific deploy project (eks-deploy-brain-tasks)

## CodeBuild Service Role For Build Project (brain-tasks)

Builds Docker images
Pushes to Docker Hub
Example: 
Role_name: codebuild-brain-tasks-service-role 
Provide Access to secrets manager secret for (GetSecretValue) Other than customer managed role.

## CodeBuild Service Role For Deploy Project (eks-deploy-brain-tasks)

Pulls image from S3 bucket
Deploy to EKS cluster

Example: 
Role_name: codebuild-eks-deploy-brain-tasks-service-role
### codebuild-eks-deploy-brain-tasks-project-access-to-s3-policy
create policy - Need Access to S3 bucket for (getobjects,listbuckets & getobjectversion) Other than customer managed policy-
### codebuild-eks-deploy-brain-tasks-service-role-access-to-eks
create policy- attach describe cluster & list-cluster to above policy needed to deploy workload on EKS.
Add these two policy to Role_name mentioned above


## EKS Node Group Role

Add EKS Node Role [eksctl-brain-cluster-nodegroup-bra-NodeInstanceRole-aYxeLgsJ1mrR] to CloudWatchAgentServerPolicy to monitor cluster 


## Kubernetes Access Role (aws-auth)

Even with above IAM permission, EKS will reject access unless the role is mapped. You must add your CodeBuild deploy role into aws-auth ConfigMap

Download configmap aws-auth to file

```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth.yaml
```
open in nano editor and paste under map role. Mention the code deploy service role



```bash
mapRoles:
  - rolearn: arn:aws:iam::187393210846:role/codebuild-eks-deploy-brain-tasks-service-role
    username: codebuild
    groups:
      - system:masters
```
reapply using 
```bash
kubectl apply -f aws-auth.yaml
```
Now aws-auth configMap is updated and now codebuild deploy role is able to access EKS cluster


## 📂 Project Structure

```

BrainTasks/
│
├── app/                         # Application + Docker
│   ├── Dockerfile
│   ├── dist/                   # React build files
│
├── cicd/                       # CI/CD pipeline config
│   ├── buildspec.yml
│   ├── buildspec-deploy.yml
│
├── k8s/                        # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│
├── screenshots/                # Proof (important for evaluation)
│   ├── pipeline.png
│   ├── pods.png
│   ├── cloudwatch.png
│   ├── app-url.png
│
├── docs/ (optional)            # Your Word/PDF explanation
│   └── Application-Deployment.docx
│
└── README.md

```

---

## Clone Repository

```bash
git clone https://github.com/Nalini-0212/Brain-Tasks.git

cd Brain-Tasks
```

---

## Docker

### Dockerfile

```dockerfile
FROM httpd:2.4-alpine
RUN rm -rf /usr/local/apache2/htdocs/*
COPY dist/ /usr/local/apache2/htdocs/
EXPOSE 80
CMD ["httpd-foreground"]
```

### Build Image

```bash
cd app
docker build -t <dockerhub-username>/braintask:latest .
```
Example

```bash
docker build -t naliniselv/braintask:latest .
```
OR

USE below command

```bash
docker build -t naliniselv/braintask:latest -f app/Dockerfile .
```
### Run Container

```bash
docker run -d --name  braintasks-container -p 3000:80 naliniselv/braintask:latest
```

👉 Serves the application using Apache on port 80

---

Open

```
http://<server-ip>:3000
```

---

### Push Image to Docker Hub

```bash
docker login

docker push naliniselv/braintask:latest
```

---

### Docker Hub Repository

Docker Image:

```text
docker.io/naliniselv/braintask:latest
```

---

## Amazon EKS

### Create EKS cluster

---

```bash
eksctl create cluster --name brain-cluster --region us-east-1 --nodegroup-name brain-ng --node-type t2.medium --nodes 2
```
---

### Configure kubectl

```bash
aws eks update-kubeconfig \
--region us-east-1 \
--name brain-cluster
```

---

### Deploy Application

```bash
kubectl apply -f k8s/deployment.yaml

kubectl apply -f k8s/service.yaml
```

---

### Verify Deployment

```bash
kubectl get nodes

kubectl get deployment

kubectl get pods

kubectl get svc
```
### Verification Output

```bash
kubectl get pods
```

```
NAME                                READY   STATUS    RESTARTS   AGE
brain-deployment-6fddf44878-cdw9k   1/1     Running   0          63s
brain-deployment-6fddf44878-hsg4f   1/1     Running   0          63s
brain-deployment-6fddf44878-swxzb   1/1     Running   0          63s
```

```bash
kubectl get svc
```

```
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP                                                              PORT(S)          AGE
brain-service   LoadBalancer   10.100.178.149   a72d01bf965164b30aac84f8817c559f-872428231.us-east-1.elb.amazonaws.com   3000:30192/TCP   10m
kubernetes      ClusterIP      10.100.0.1       <none>                                                                   443/TCP          22m
```

---

## CI/CD Pipeline

The project uses AWS CodePipeline to automate deployments.

Pipeline Flow

```
GitHub
   │
   ▼
CodePipeline
   │
   ▼
CodeBuild
   │
   ├── Build Docker Image
   ├── Push Image to Docker Hub
   ├── Update Kubernetes Manifest
   └── Deploy to Amazon EKS
```

## CI/CD Pipeline Explanation

### Stage 1 – Source

- Source code is retrieved from the GitHub repository through AWS CodePipeline.

### Stage 2 – Build (AWS CodeBuild)

- Builds the Docker image from the `app/` directory.
- Tags the Docker image using the build number (`V1.<CODEBUILD_BUILD_NUMBER>`).
- Pushes the image to Docker Hub.
- Generates `versions.txt` and `image.txt`.

### Stage 3 – Deploy

- Updates the Kubernetes deployment.
- Applies the Kubernetes manifests.
- Performs a rolling update with zero downtime.

```bash
kubectl apply -f k8s/deployment.yaml

kubectl apply -f k8s/service.yaml

kubectl set image deployment/brain-deployment \
brain-container=$IMAGE_URL

kubectl rollout status deployment/brain-deployment
```

👉 Ensures zero-downtime rolling updates

## Kubernetes Resources

### Deployment

- Replicas: 3
- Container Image: Docker Hub
- Container Port: 80

### Service

- Type: LoadBalancer
- Service Port: 80
- Target Port: 80
---

## Application URL

http://a3622c24c46a6442cbde66803f36fa12-709266722.us-east-1.elb.amazonaws.com

---
## Amazon EKS Cluster

**Cluster Name**

brain-cluster

**Cluster ARN**

arn:aws:eks:us-east-1:187393210846:cluster/brain-cluster

---

## AWS LoadBalancer ARN

arn:aws:elasticloadbalancing:us-east-1:187393210846:loadbalancer/a3622c24c46a6442cbde66803f36fa12

---

## AWS CodeBuild

CodeBuild performs the following tasks:

- Downloads source code
- Builds Docker image
- Logs in to Docker Hub
- Pushes Docker image
- Updates Kubernetes deployment manifest
- Deploys application to Amazon EKS using kubectl

Build specification file:

```
cicd/buildspec.yml
```
Sample File:

```

version: 0.2
env:
  variables:
    IMAGE_REPOSITORY_NAME: "naliniselv/braintask"
  secrets-manager:
    DOCKERHUB_USERNAME: "dockerhubcred:username"
    DOCKERHUB_PASSWORD: "dockerhubcred:password"

phases:
  pre_build:
    commands:
      - "echo Logging in to DockerHub..."
      - "echo $DOCKERHUB_PASSWORD | docker login -u $DOCKERHUB_USERNAME --password-stdin"
      - "export VERSION=V1.$CODEBUILD_BUILD_NUMBER"
      - "export IMAGE_URL=$IMAGE_REPOSITORY_NAME:$VERSION"
      - "echo Image_URL: '$IMAGE_URL'"
      - "echo Build_Version: '$VERSION'"
  build:
    commands:
      - "echo Building the Docker image....."
      - "docker build -t $IMAGE_REPOSITORY_NAME:$VERSION -f app/Dockerfile ./app"
      - "echo Tagging the Docker image....."
      - "docker tag $IMAGE_REPOSITORY_NAME:$VERSION $IMAGE_REPOSITORY_NAME:latest"
  
  post_build:
    commands:
      - "echo Pushing the docker images with versions....."
      - "docker push $IMAGE_URL"
      - "echo Pushing the docker image with latest tag"
      - "docker push $IMAGE_REPOSITORY_NAME:latest"
      - "echo saving the version information to file"
      - "echo $VERSION > versions.txt"
      - "echo $IMAGE_URL > image.txt"
    

  
artifacts:
  files:
    - 'versions.txt'
    - 'image.txt'
    - 'k8s/deployment.yaml'
    - 'k8s/service.yaml'
    - 'cicd/buildspec-deploy.yml'
```


Deploy specification file:

```
cicd/buildspec-deploy.yml
```
Sample File:

```
version: 0.2
env:
  variables:
    CLUSTER_NAME: 'brain-cluster'
    AWS_DEFAULT_REGION: 'us-east-1'
phases:
  install:
    commands:
      - "set -e"
      - "echo Installing kubectl...."
      - "curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
      - "curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
      - "echo $(cat kubectl.sha256)  kubectl | sha256sum --check"
      - "chmod +x kubectl"
      - "mkdir -p ~/.local/bin"
      - "mv ./kubectl ~/.local/bin/kubectl"
      - "export PATH=$PATH:/root/.local/bin"
  pre_build:
    commands:
      - "echo Cluster Name: '$CLUSTER_NAME'"
      - "echo Region: '$AWS_DEFAULT_REGION'"
      - "aws sts get-caller-identity"
      - "aws eks list-clusters --region $AWS_DEFAULT_REGION"
      - "echo checking contents"
      - "ls -ltr"
      - "echo Loading version info..."
      - "export VERSION=$(cat versions.txt)"
      - "export IMAGE_URL=$(cat image.txt)"
      - "echo Deploying Version: '$VERSION'"
      - "echo Image URL: '$IMAGE_URL'"
      - "echo Updating kubeconfig for the cluster ...."
      - "aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name $CLUSTER_NAME"
      - "echo checking the nodes in the cluster......"
      - "kubectl get nodes"

  build:
    commands:
      - "echo Applying manifests files to the cluster....."
      - "kubectl apply -f k8s/deployment.yaml"
      - "kubectl apply -f k8s/service.yaml"
      - "echo Updating image dynamically..."
      - "kubectl set image deployment/brain-deployment brain-container=$IMAGE_URL"

  post_build:
    commands:
      - "echo Checking the deployments in the cluster....."
      - "kubectl get deployments"
      - "echo Checking deployed image..."
      - "kubectl describe deployment brain-deployment | grep Image"
      - "echo getting status of pods in the cluster....."
      - "kubectl get pods"
      - "echo checking the rollout status of the deployment....."
      - "kubectl rollout status deployment/brain-deployment"
      - "echo Waiting additional 20 seconds for stability..."
      - "sleep 20"
      - "echo Final pod status:"
      - "kubectl get pods -o wide"
      - "echo getting pod logs"
      - "kubectl logs -l app=brain"
      - "echo getting services in the cluster....."
      - "kubectl get svc brain-service"
      - "echo Deployment and Service created successfully"

```

---

## Monitoring

AWS CloudWatch Logs are used to monitor:

- CodeBuild execution
- Deployment logs
- Build errors
- Kubernetes deployment status

## CloudWatch Monitoring

CloudWatch is used to monitor the complete CI/CD workflow.

### Build Logs

```
/aws/codebuild/brain-tasks
```

### Deployment Logs

```
/aws/codebuild/eks-deploy-brain-tasks
```

### Application Logs

```
/aws/containerinsights/brain-cluster/application
```

Container Insights is enabled using the Amazon CloudWatch Observability add-on to collect Kubernetes metrics and application logs.

👉 Logs collected using:

Fluent Bit
CloudWatch Agent


📊 Container Insights

```bash
aws eks create-addon \
  --cluster-name brain-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-1
```

## Verification Commands

```bash
kubectl get pods

kubectl get svc

kubectl logs -l app=brain
```

---

## Useful Commands

Docker

```bash
docker images #to check docker images

docker ps #to check running containers

docker ps -a #to check all containers

docker stop <container-name> #to stop particular container

docker rm <container-name> #to remove container

docker rmi <image-name> #to remove image
```

Kubernetes

```bash
kubectl get nodes

kubectl get pods

kubectl get deployments

kubectl get svc

kubectl describe deployment brain-deployment

kubectl logs <pod-name>
```

---

# 📸 Screenshots
The following screenshots are included in the `screenshots/` folder:

| Screenshot | Description |
|------------|-------------|
| GitHub Repository | Source code repository |
| Docker Hub | Docker image repository |
| Amazon EKS | Running Kubernetes cluster |
| Kubernetes Pods | Running application pods |
| Kubernetes Service | LoadBalancer service |
| CodeBuild | Successful build execution |
| CodePipeline | Successful CI/CD pipeline |
| CloudWatch Logs | Build and deployment logs |
| Application | Running application in browser |

---

## Key Features

- End-to-end CI/CD pipeline using AWS CodePipeline
- Dockerized React application
- Docker Hub image repository
- Deployment on Amazon EKS
- Kubernetes LoadBalancer service
- Rolling updates with zero downtime
- CloudWatch Logs monitoring
- CloudWatch Container Insights

---

## Author

**Nalini Selvaraj**

- **GitHub Profile:** <https://github.com/Nalini-0212>
- **Repository:** <https://github.com/Nalini-0212/Brain-Tasks>
---

## Conclusion

This project demonstrates a complete DevOps CI/CD workflow for deploying a React application using Docker, Kubernetes, Amazon EKS, AWS CodeBuild, and AWS CodePipeline. The pipeline automates building, publishing, and deploying the application while CloudWatch provides centralized logging and monitoring.

---

## Project Status

**Status:** Completed

This project successfully demonstrates a complete CI/CD pipeline for deploying a Dockerized React application on Amazon EKS using AWS CodePipeline and CodeBuild. Monitoring is enabled through Amazon CloudWatch Logs and Container Insights.

---




















