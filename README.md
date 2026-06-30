# End-to-End-CI-CD-Pipeline-using-AWS-CodePipeline-CodeBuild-ECS
## AWS Services Used

- GitHub
- AWS CodeConnections
- AWS CodePipeline
- AWS CodeBuild
- Amazon ECS
- AWS Systems Manager Parameter Store
- Docker
- Docker Hub
## Step 1: Clone Repository

git clone https://github.com/jeevitha-y/aws-dockerized-django-web-application.git
 cd aws-dockerized-django-web-application

---
## Step 2: Store Docker Credentials in Parameter Store

/docker/registry/username  -> String
/docker/registry/password  -> SecureString

---
## Step 3: Create CodeBuild Project

Source: GitHub (CodeConnections)
Repo: jeevitha-y/aws-dockerized-django-web-application
Branch: main

Environment:
- Ubuntu
- Python 3.11
- Privileged mode enabled (Docker required)

---
## IAM Permissions (CodeBuild Role)

- AmazonSSMFullAccess

{
  "Effect": "Allow",
  "Action": [
    "codeconnections:GetConnection",
    "codeconnections:GetConnectionToken",
    "codeconnections:UseConnection"
  ],
  "Resource": "*"
}

## Step 4: Create CodePipeline

Stages:

Source → Build → Deploy (ECS)

---
## Source Stage

GitHub (CodeConnections)
Branch: main

---

## Build Stage (CodeBuild)

What happens:

- Docker login using Parameter Store
- Docker image build
- Docker push to Docker Hub
- imagedefinitions.json created

---
## Step 5: buildspec.yml
```
version: 0.2

env:
  variables:
    DOCKER_REGISTRY_URL: "Docker.io"
  parameter-store:
    DOCKER_REGISTRY_USERNAME: "/docker/registry/username"
    DOCKER_REGISTRY_PASSWORD: "/docker/registry/password"

phases:
  install:
    runtime-versions:
      python: 3.11

  pre_build:
    commands:
      - echo "$DOCKER_REGISTRY_PASSWORD" | docker login -u "$DOCKER_REGISTRY_USERNAME" --password-stdin

  build:
    commands:
      - cd apps/django-app
      - docker build -t $DOCKER_REGISTRY_USERNAME/python_django:latest .
      - docker push $DOCKER_REGISTRY_USERNAME/python_django:latest

  post_build:
    commands:
      - cd $CODEBUILD_SRC_DIR
      - printf '[{"name":"python-app","imageUri":"%s/python_django:latest"}]' "$DOCKER_REGISTRY_USERNAME" > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```
---
## Step 6: Create ECS Resources

### 1. Create an ECS Cluster

- Open the AWS ECS Console.
- Click **Create Cluster**.
- Choose the required infrastructure (Fargate or EC2).
- Enter a cluster name.
- Create the cluster.

---
### 2. Create a Task Definition

- Go to **Task Definitions**.
- Click **Create new Task Definition**.
- Choose the launch type (Fargate or EC2).
- Configure:
  - Task Definition Family
  - CPU and Memory
  - Task Execution IAM Role
- Add a container:
  - Container Name: `python-app`
  - Image URI: `docker.io/<dockerhub-username>/python_django:latest`
  - Container Port: `8000` (or your application's port)
- Save the task definition.

> **Note:** The container name (`python-app`) must match the `name` field in `imagedefinitions.json`.

---
### 3. Create an ECS Service

- Open your ECS Cluster.
- Click **Create Service**.
- Select the Task Definition created above.
- Enter a Service Name.
- Choose the desired number of tasks.
-   Add an inbound rule to the Security Group:
    - **Type:** Custom TCP
    - **Port:** 8000
    - **Source:** Anywhere (0.0.0.0/0) *(for testing)*

> **Note:** For production environments, restrict the source to only trusted IP addresses or an Application Load Balancer (ALB).

- Create the service.

---
## Step 7: Add ECS Deploy Stage to CodePipeline

Add a new stage named **Deploy**.

Provider:

```
Amazon ECS
```

Configure:

- Cluster: Your ECS Cluster
- Service: Your ECS Service
- Input Artifact: Build Artifact
- Image definitions file: `imagedefinitions.json`
## CodePipeline Service Role Permissions

For this project, the CodePipeline service role was granted:

- AmazonECS_FullAccess

This allows CodePipeline to deploy the latest Docker image to Amazon ECS during the Deploy stage.

> **Note:** In production environments, it is recommended to grant only the minimum required ECS and IAM permissions instead of full access.
