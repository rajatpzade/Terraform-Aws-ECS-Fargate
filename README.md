# Terraform-Aws-ECS-Fargate

# Step 1: Set Up the Hello World Node.js App
## 1.1 Create the Node.js App
	1	Initialize a new Node.js project:
```
mkdir hello-world-nodejs
cd hello-world-nodejs
npm init -y
```

## Create the index.js file:

```
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => res.send('Hello World!'));

app.listen(port, () => console.log(`App listening at http://localhost:${port}`));
```


 ## Install Express:

```
npm install express
```

## 1.2 Create a Dockerfile
	1	Create a Dockerfile in the project directory
```
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

## 1.3 Test the Docker Image Locally
	
    1	Build and run the Docker image
```
docker build -t hello-world-nodejs .
docker run -p 3000:3000 hello-world-nodejs
```

	◦	Open your browser and go to http://localhost:3000 to see if the app is running.

## Step 2: Set Up Terraform Configuration for AWS ECS/Fargate

#### 2.1 Set Up the Directory Structure

Create a new directory for your Terraform configuration:
 ```
 mkdir terraform
cd terraform
```
#### 2.2 Write Terraform Configuration Files

2.2.1 Provider Configuration (provider.tf)
```
provider "aws" {
  region = "us-east-1"
}
```

2.2.2 ECS Cluster and Fargate Service (ecs.tf)

```
resource "aws_ecs_cluster" "hello_world_cluster" {
  name = "hello-world-cluster"
}

resource "aws_ecs_task_definition" "hello_world_task" {
  family                   = "hello-world-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn

  container_definitions = jsonencode([{
    name      = "hello-world"
    image     = "your_dockerhub_username/hello-world-nodejs"
    essential = true
    portMappings = [{
      containerPort = 3000
      hostPort      = 3000
    }]
  }])
}

resource "aws_ecs_service" "hello_world_service" {
  name            = "hello-world-service"
  cluster         = aws_ecs_cluster.hello_world_cluster.id
  task_definition = aws_ecs_task_definition.hello_world_task.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = [aws_subnet.public_subnet_1.id, aws_subnet.public_subnet_2.id]
    security_groups  = [aws_security_group.ecs_security_group.id]
    assign_public_ip = true
  }
}
```
2.2.3 IAM Roles and Policies (iam.tf)

```
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```
2.2.4 VPC and Networking (network.tf)
```
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public_subnet_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
}

resource "aws_subnet" "public_subnet_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  map_public_ip_on_launch = true
}

resource "aws_security_group" "ecs_security_group" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```
#### 2.3 Initialize and Apply Terraform Configuration

Initialize Terraform: 
```
terraform init
```
Apply Terraform Configuration:
```
terraform apply -auto-approve
```
#### Step 3: Set Up GitHub Repository and GitHub Actions
3.1 Initialize a GitHub Repository
Create a new public repository on GitHub (e.g., hello-world-nodejs-ecs).

Push your Node.js app and Terraform configuration to the repository

```
git init
git remote add origin <your-github-repo-url>
git add .
git commit -m "Initial commit"
git push -u origin main
```
3.2 Set Up GitHub Actions for Continuous Deployment
3.2.1 Create a GitHub Actions Workflow
Create a .github/workflows/deploy.yml file in your repository:
yaml

```
name: CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Set up Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '14'

    - name: Install dependencies
      run: npm install

    - name: Build Docker image
      run: docker build -t your_dockerhub_username/hello-world-nodejs .

    - name: Login to DockerHub
      run: echo ${{ secrets.DOCKERHUB_PASSWORD }} | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

    - name: Push Docker image
      run: docker push your_dockerhub_username/hello-world-nodejs

    - name: Set up Terraform
      uses: hashicorp/setup-terraform@v1

    - name: Terraform Init
      run: terraform init
      working-directory: ./terraform

    - name: Terraform Apply
      run: terraform apply -auto-approve
      working-directory: ./terraform
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```
3.2.2 Set Up GitHub Secrets
Add the following secrets to your GitHub repository:
- DOCKERHUB_USERNAME
- DOCKERHUB_PASSWORD
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

#### Step 4: Create a Screencast Demonstration

1.Record Your Screen:

- Use a screen recording tool (e.g., OBS Studio) to record the setup and deployment process.
- Demonstrate the Node.js app running on AWS ECS/Fargate.
Upload the Screencast:

2. Upload the screencast to a video sharing platform (e.g., YouTube) or provide the video file.

#### Step 5: Deliver the Project
Share the GitHub Repository:

Provide the link to your public GitHub repository.
Share the Screencast:

Provide the link to the screencast demonstration