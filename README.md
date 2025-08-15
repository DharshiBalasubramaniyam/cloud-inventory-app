# Inventory Management System – Cloud-Native Full Stack Application

A production-grade inventory management application designed with a modern, scalable, and secure architecture.

## Tech Stack

<img width="4237" height="1846" alt="image" src="https://github.com/user-attachments/assets/239f31ec-b50e-4b94-bb65-efe1fa46535b" />

- 💻 [Web-tier](#web-tier):
  - **React.js** with Vite for blazing-fast development and optimized builds.
- ⚙️ [Logic-tier](#logic-tier):
  - **Spring Boot (Java)** REST API handling all CRUD operations.
- 🗄 Database:
  - **MongoDB Atlas** – cloud-hosted, highly available NoSQL database.
- 📦 [Containerization](#containerization):
  - Application components (web-tier and logic-tier) are packaged as **Docker** images for consistent, portable deployments.
- 📁 [Container registry](#container-registry-ecr):
  - Docker images are stored in **Amazon Elastic Container Registry (ECR)**.
- 🌐 [Networking](#networking):
  - [**AWS VPC**](#vpc) with both public and private subnets across 2 Availability Zones.
  - [**Public subnets**](#subnets) host the web-tier tasks. 
  - [**Private subnets**]((#subnets)) host the logic-tier tasks.
  - [**Load balancers**](load-balancing) to distribute traffic across subnets.
  - [**NAT Gateway**](#nat-gateway) enables secure outbound internet access from private subnets (logic-tier).
  - [**Route tables**](#route-tables):
    - Public subnet route tables direct 0.0.0.0/0 traffic to the [Internet Gateway](internet-gateway).
    - Private subnet route tables direct 0.0.0.0/0 traffic to the NAT Gateway.
    - Both subnets have default route for subnet to subnet communication within the VPC.
  - [**Security Groups**](security-groups) tightly control inbound and outbound traffic between services and ALBs.
- 🚀 [Deployment](#deployment):
  - Deployed to **AWS ECS (Fargate)** in a multi-AZ setup for high availability.
- 🔄 [CI/CD](#ci-cd-with-github-actions):
  - Automated pipeline using **GitHub Actions**:
    - Build and test the application on code push.
    - Package the application as a Docker image and push to Amazon ECR.
    - Update ECS task definition with new image.
    - Deploy updated task definition to ECS.

## Web-tier 

- React.js for building a responsive, component-based UI.
- Vite as the build tool, providing lightning-fast Hot Module Replacement (HMR) and minimal bundle sizes.
- Tailwind CSS for styling.
- React Context API for managing global state, ensuring seamless data sharing across components without excessive prop drilling.
- The application contains a single route (/) that serves as the main dashboard, which displays a list of inventory details and provides UI interactions such as viewing, adding, editing, and deleting inventory records.

## Logic-tier 

- The backend is implemented using Spring Boot, providing a RESTful API for all inventory management operations. It communicates with MongoDB Atlas to store and retrieve inventory records.

| HTTP Method | Route Path | Parameters | Description |  
|----------|----------|----------|----------| 
| <img alt="Static Badge" src="https://img.shields.io/badge/post-green?style=for-the-badge"> | `/inventory`   | - | Create new inventory record | 
| <img alt="Static Badge" src="https://img.shields.io/badge/put-yellow?style=for-the-badge"> | `/inventory`   | inventoryId | Edit inventory record | 
| <img alt="Static Badge" src="https://img.shields.io/badge/delete-red?style=for-the-badge"> | `/inventory`   | inventoryId | Delete existing inventory record |
| <img alt="Static Badge" src="https://img.shields.io/badge/get-blue?style=for-the-badge"> | `/inventory`   | - | Get all inventories |
| <img alt="Static Badge" src="https://img.shields.io/badge/get-blue?style=for-the-badge"> | `/inventory/{inventoryId}`   | - | Get inventory by id |

## Containerization

- The Web tier and Logic tier are containerized using multi-stage build, where the first stage (build-stage) compiles the code and generates build artifacts, and the second stage (runtime-stage) creates the final, production-ready image using build artifacts from the first stage.
  
### Web tier Dockerfile

```dockerfile
# build-stage
FROM node:18-alpine AS builder

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

RUN npm run build

# runtime-stage
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Logic tier Dockerfile

```dockerfile
# build-stage
FROM maven:3.9.6-eclipse-temurin-21 AS builder

WORKDIR /app

COPY pom.xml .

COPY src ./src

RUN mvn clean package -DskipTests

# runtime-stage
FROM openjdk:21-jdk-slim

WORKDIR /app

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```

## Container Registry (ECR)

- The project uses Amazon Elastic Container Registry (ECR) as the secure, fully managed container image repository for storing application images, which integrate well with AWS ECS.

| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-tier-registry`   | `inventory-logic-tier-registry` | 
| Region | `us-east-1`   | `us-east-1` | 
| Visibility | `private`   | `private` | 

- Since both ECR repositories are private, they are accessible only to services which has an authorized AWS [IAM role](#iam-role-for-ecs-services).

## Networking

<img width="2283" height="1352" alt="image" src="https://github.com/user-attachments/assets/248d4395-9199-4f02-ae4d-2711c838eb7e" />

### VPC

- VPC is a logically isolated virtual network within the cloud dedicated to AWS, where users can launch their services.
- Create a VPC to deploy our applications in the AWS cloud.

| Configuration | Value | 
|----------|----------|
| Name | `inventory-vpc` |  
| Region | `us-east-1` | 
| CIDR block | `10.0.0.0/16` | 

### Subnets

- A region is a geographical location that contains multiple Availability Zones. E.g., `us-east-1`
- An availability zone is an independent data center inside a region. E.g., `us-east-1a`, `us-east-1b`, `us-east-1c`, `us-east-1d`
- A subnet is a range of IP addresses within a VPC, hosted inside a single AZ. E.g., If VPC's CIDR block is `10.0.0.0/16`, the subnets can be `10.0.0.0/24`, `10.0.1.0/24`, `10.0.2.0/24`, and so on.
  - In this example, VPC has `2^16 = 65536` IP addresses.
  - With subnet mask `/24`, one subnet can contain `2^8 = 256` IP addresses.
  - This configuration allows `65536/256 = 256` subnets in the VPC.
- A subnet belongs to exactly one AZ (cannot span multiple AZs).
- A subnet can be public (Internet-accessible via an Internet Gateway) or private (no direct Internet access).
- My case:
  - Application deployed across two AZs for fault tolerance—if one AZ goes down, the other continues serving traffic.
  - 2 private subnets (for the logic tier) and 2 public subnets (for the web tier).
  - Each AZ contains 1 public subnet + 1 private subnet.

| Configuration | Subnet 1 | Subnet 2 | Subnet 3 | Subnet 4 |  
|----------|----------|----------|----------| ----------|
| VPC | `inventory-vpc`   | `inventory-vpc` | `inventory-vpc` | `inventory-vpc` |
| Availability zone | `us-east-1a`   | `us-east-1a` | `us-east-1b` | `us-east-1b` | 
| Visibility | `public`   | `private` | `public` | `private` |
| Name | `inventory-public1-subnet-1a`   | `inventory-private1-subnet-1a` | `inventory-public2-subnet-1b` | `inventory-private2-subnet-1b` |
| CIDR block | `10.0.0.0/24`   | `10.0.1.0/24` | `10.0.2.0/24` | `10.0.3.0/24` |

### Internet Gateway

- Internet Gateway allows internet traffic to reach the public subnets. E.g., User accesses web application via web browser.
- Internet Gateway is attached to the VPC.

| Configuration | Value | 
|----------|----------|
| Name | ` inventory-igw` | 
| VPC | `inventory-vpc` |  

### NAT Gateway

- Private subnets deny internet traffic that is routed from the public internet.
- A NAT Gateway  in AWS is a managed service that allows resources in a private subnet to connect to the internet without exposing them to inbound internet traffic.
- A NAT Gateway provides outbound internet access only, not inbound internet traffic.
- The NAT gateway is placed inside a public subnet.
- The NAT gateway is associated with a unique Elastic IP address, which is public.
- How does it work?
  - An instance in the private subnet sends a request to the Internet.
  - The request is routed to the NAT Gateway (in a public subnet).
  - The NAT Gateway replaces the private IP with its public Elastic IP.
  - The response from the Internet is sent back to the NAT Gateway, which translates it back to the private IP and returns it to the instance
- In our case, we need a NAT gateway because the logic-tier (which is in private subnets) needs to access MongoDB Atlas over the internet.

| Configuration | Value | 
|----------|----------|
| Name | ` inventory-nat-1a` | 
| subnet | `10.0.0.0/24` |  
| Elasic IP | Allocate Elastic IP |  

### Route tables

- Each subnet must be attached to a route table.
- A route table defines how packets are routed out of the subnet.
- When creating a route table it automatically creates a route (default), which lets subnets in the VPC talk to each other.
- Here, both public subnets have similar route tables (`inventory-web-rt`), and both private subnets have similar route tables (`inventory-web-rb`).
- Other than the default route,
  - Public subnet must have a route to the internet gateway.
  - Private subnets must have a route to the NAT Gateway.

| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-rt`   | `inventory-logic-rt` | 
| VPC | `inventory-vpc`   | `inventory-vpc` | 
| Subnets | `inventory-public1-subnet-1a`, `inventory-public2-subnet-1b`   | `inventory-private1-subnet-1a`, `inventory-private2-subnet-1b` | 

Route table for web tier: `inventory-web-rt`

| Destination | Target | 
|----------|----------|
| `10.0.0.0/16` | `local`   | 
| `0.0.0.0/0` | `inventory-igw`   | 

Route table for logic tier: `inventory-logic-rt`

| Destination | Target | 
|----------|----------|
| `10.0.0.0/16` | `local`   | 
| `0.0.0.0/0` | `inventory-nat-1a`   | 

### Security Groups

- A way to filter inbound and outbound traffic that allows access to the instances (e.g., EC2 instance, ECS service, RDS instance) in a VPC.
- A security group is a firewall that acts at the instance level, not the subnet level.
  - An inbound rule defines what incoming traffic is permitted to reach the associated AWS resource.
  - An outbound rule determines what outgoing traffic is allowed to leave the associated AWS resource.
- SGs are stateful, meaning for every inbound rule, an outbound rule is allowed and vice versa.
- We apply SGs to AWS resources to control inbound and outbound access:
  - **Web ALB** – Routes external traffic to the web-tier ECS tasks.
  - **Web-tier ECS service** – Runs the web-tier containers.
  - **Logic ALB** – Routes internal traffic to the logic-tier ECS tasks.
  - **Logic-tier ECS service** – Runs the logic-tier containers.
- Security Group Design:
  - The web ALB is the only entry point from the internet.
  - The web service can only be reached through the web ALB.
  - The logic service can only be reached through the logic ALB.

Security group for web ALB: `inventory-web-alb-sg`

- Inbound rule:
  
| Type | Source | Protocol | Port range | 
|-----------|----------|----------|----------|
| HTTP | `0.0.0.0/0` | `TCP`   | `80` | 

- Outbound rule:
  
| Type | Destination | Protocol | Port range | 
|-----------|----------|----------|----------|
| HTTP | `0.0.0.0/0` | `TCP`   | `80` |

Security group for web-tier ECS service: `inventory-web-sg`

- Inbound rule:
  
| Type | Source | Protocol | Port range | 
|----------|----------|----------|----------|
| HTTP | `inventory-web-alb-sg` | `TCP`   | `80` | 

- Outbound rule:
  
| Type | Destination | Protocol | Port range | 
|-----|-----|----------|----------|
| HTTP | `0.0.0.0/0` | `TCP`   | `80` |
| HTTPS | `0.0.0.0/0` | `TCP`   | `443` |

Security group for logic ALB: `inventory-logic-alb-sg`

- Inbound rule:
  
| Type | Source | Protocol | Port range | 
|--|---------|----------|----------|
| HTTP | `0.0.0.0/0` | `TCP`   | `80` |

- Outbound rule:
  
| Type | Destination | Protocol | Port range | 
|----|------|----------|----------|
| `All traffic` | `inventory-logic-sg` | `all`   | `all` |

Security group for logic-tier ECS service: `inventory-logic-sg`

- Inbound rule:
  
| Type | Source | Protocol | Port range | 
|--------|--|----------|----------|
| `Custom TCP` | `inventory-logic-alb-sg` | `TCP`   | `8080` |

- Outbound rule:
  
| Type | Destination | Protocol | Port range | 
|----|------|----------|----------|
| `All traffic` | `0.0.0.0/0` | `all`   | `all` |

## Load Balancing

- Load balancing is the process of distributing incoming network traffic across multiple resources (like servers, ECS tasks, or containers) so no single resource is overwhelmed.
- Types of load balancing:
  - Load balancing within one service: Distributes incoming requests evenly across multiple instances/tasks of the same service. Example: You have 5 ECS tasks running the web-tier 
  - Load balancing across multiple services (routing): Routes traffic to different services based on rules (e.g., path or hostname). Example: `/product/*` → Product service, `/auth/*` → Auth service
- We do not require path-based or host-based routing. We only need to distribute traffic evenly across multiple tasks of a particular ECS service. Because we are going to run 2 tasks for each service across availability zones.
- Web ALB: Accepts requests from the internet and distributes them to multiple web-tier ECS tasks.
- Logic ALB (for logic tier): Accepts requests from the web tier and distributes them to multiple logic-tier ECS tasks.
- Each ALB is dedicated to distributing traffic evenly across multiple tasks of its respective ECS service.


### Target groups

- A target group is a logical grouping of multiple instances/tasks of the same service that the ALB sends traffic to.
- Each ECS service usually has its own target group, which distributes incoming requests evenly across its tasks.

| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-tg`   | `inventory-logic-tg` | 
| Type | `IP address`   | `IP address` | 
| VPC | `inventory-vpc`   | `inventory-vpc` | 
| Protocol & port | `http`, `80` | `http`, `8080` |
| Heath check | `/index.htm;` | `/inventory` |

### Application Load Balancer (ALB)

- ALB distributes incoming traffic evenly across multiple targets within a single target group.
  
| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-alb`   | `inventory-logic-alb` | 
| Type | `internet-facing`   | `internet-facing` | 
| VPC | `inventory-vpc`   | `inventory-vpc` | 
| Subnets | `inventory-public1-subnet-1a`, `inventory-public2-subnet-1b`   | `inventory-public1-subnet-1a`, `inventory-public2-subnet-1b` |
| Security Group | `inventory-web-alb-sg` | `inventory-logic-alb-sg` |
| Listener | `http`, `80` | `http`, `80` |
| Target group | `inventory-web-tg` | `inventory-logic-tg` |

- Now our React application should send to api requests to `inventory-logic-alb`. 
- Select `inventory-logic-alb` and copy the DNS name. 
- Update `API_BASE_URL` in [`ApiConfig.tsx`](web-tier/src/api-service/ApiConfig.jsx) as below.

```
export const API_BASE_URL = "http://YOUR_ALB_DNS_NAME";
```

## Deployment

<img width="4750" height="2321" alt="image" src="https://github.com/user-attachments/assets/86df9bc3-2df4-4983-bcd9-87b3453696e8" />

- Both the web-tier and the logic tier are deployed to  **AWS ECS (Fargate)**, which is a fully managed container orchestration service provided by Amazon Web Services (AWS).
  
### IAM role for ECS services 

- Our ECR repositories are private, so only authenticated services can access them.
- To enable access, we create an IAM role with a policy granting permissions to pull from ECR.
- We attach this role to our ECS services, allowing them to access the private ECR repositories.
- `AmazonECSTaskExecutionRolePolicy` is an AWS-managed policy that can be attached to IAM users, roles, or groups in your account.
- This policy grants permissions to pull container images from Amazon ECR and send logs to Amazon CloudWatch.
- Since this policy already exists, we only need to create a role with the following configuration. We will attach this role to the Task definitions later.

| Configuration | Value | 
|----------|----------|
| Trusted Entity Type | `AWS service` | 
| Service | `Elastic Container Service` |  
| Use case | `Elastic Container Service Task` |  
| Permission policy | From the list select `AmazonECSTaskExecutionRolePolicy` |  
| name | `ecsTaskExecutionRole` |  
| Description | `Allows ECS tasks to call AWS services on your behalf.` | 
| Trusted policy | Leave the default | 

> A Trust Policy is a JSON document that defines who (which principal) is allowed to assume the role. Here, the ECS Task can assume the role `ecsTaskExecutionRole`

> A Permission Policy is a JSON document that defines what actions the role can perform. Here, `ecsTaskExecutionRole` can pull images from ECR registries.

### Task definition

- A blueprint that describes how a Docker container should launch. It includes settings like the Docker image to use, CPU and memory requirements, and environment variables.
- Task is the instantiation of a task definition.
  
| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-task-definition`   | `inventory-logic-task-definition` | 
| Launch type | `Fargate`   | `Fargate` | 
| IAM role | `ecsTaskExecutionRole`   | `ecsTaskExecutionRole` | 
| Container name | `inventory-web-container`   | `inventory-web-container` | 
| Container Image  | URI of `inventory-web-tier-registry`   |  URI of `inventory-logic-tier-registry`  | 
| Container Port mapping | `80:80`   | `8080:8080` | 
| Env variables | - | `SPRING_DATA_MONGODB_URI`, `SPRING_DATA_MONGODB_DATABASE` |

### ECS service

- ECS service runs and maintains a specified number of tasks of a particular task definition simultaneously.
- ECS cluster is a logical grouping of ECS services.
- In our case, we create one ECS cluster and 2 ECS services for web-tier and logic-tier within that cluster. Each service maintains 2 tasks. 

ECS Cluster set-up:

| Configuration | Value | 
|----------|----------|
| Name | ` inventory-cluster` | 
| template | `Networking only (for Fargate)` |  

ECS services set-up:

| Configuration | Web tier | Logic tier | 
|----------|----------|----------|
| Name | `inventory-web-service`   | `inventory-logic-service` |
| Task definition | `inventory-web-task-definition`   | `inventory-logic-task-definition` |
| Launch type | `Fargate`   | `Fargate` |
| Cluster | `inventory-cluster`   | `inventory-cluster` |
| Desired tasks | `2`   | `2` |
| VPC | ` inventory-vpc`   | ` inventory-vpc` |
| Subnets | `inventory-public1-subnet-1a`, `inventory-public2-subnet-1b`   | `inventory-private1-subnet-1a`, `inventory-private2-subnet-1b` |
| Security groups | `inventory-web-sg`   | `inventory-logic-sg` |
| ALB | `inventory-web-alb` | `inventory-logic-alb` |

> Launch Type: Refers to how we want to deploy and manage your containers. AWS ECS supports two launch types: EC2 Launch Type, Fargate Launch Type. With EC2, we have to manage the EC2 instances(virtual servers) where our containers run. But with Fargate, AWS manages the infrastructure for us automatically, so we don’t need to provision or manage EC2 instances directly.

## CI-CD with GitHub Actions

<img width="7421" height="2480" alt="image" src="https://github.com/user-attachments/assets/79b01658-1e38-4b06-9f11-6fd8fc2bd45e" />

- Automated build, Dockerization, and deployment to AWS using GitHub Actions. Every push to the main branch triggers the CI/CD pipeline.
- To access the AWS ECS service from GitHub Actions, we need to create an IAM user within our account and provide that user's credentials to GitHub Actions so that GH has access to AWS services.

### IAM user for GitHub Actions   

- Go to IAM policies and click on `create policy` and select JSON option. Copy the content below and create a policy with the name `inventory-ga-policy`. 

```
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "PushImagesToECR",
			"Effect": "Allow",
			"Action": [
				"ecr:GetAuthorizationToken",
				"ecr:BatchCheckLayerAvailability",
				"ecr:CompleteLayerUpload",
				"ecr:InitiateLayerUpload",
				"ecr:PutImage",
				"ecr:UploadLayerPart"
			],
			"Resource": "*"
		},
		{
			"Sid": "ECSDeployPermissions",
			"Effect": "Allow",
			"Action": [
				"ecs:DescribeServices",
				"ecs:DescribeTaskDefinition",
				"ecs:RegisterTaskDefinition",
				"ecs:UpdateService"
			],
			"Resource": "*"
		},
		{
			"Effect": "Allow",
			"Action": "iam:PassRole",
			"Resource": "arn:aws:iam::019847571531:role/ecsTaskExecutionRole"
		}
	]
}

```  
- Go to IAM users and click on `create user`

| Configuration | Value | 
|----------|----------|
| Username | `inventory-ga-user` | 
| Set permissions | `Attach policies directly` |  
| Set permissions | `inventory-ga-policy` |  

- After creating a user with the above configuration, create security credentials for the user by choosing the use case `Third-party service`.

### GitHub repository secrets

- Go to your repo → Settings → Secrets and variables → Actions
- Create secrets below:
  
| Secret key | Value | 
|----------|----------|
| AWS_ACCESS_KEY_ID | access key of IAM user `inventory-ga-user`  |  
| AWS_SECRET_ACCESS_KEY | secret access key of IAM user `inventory-ga-user` | 
| AWS_REGION | `us-east-1` | 
| ECR_LOGIC_REPOSITORY | `inventory-logic-tier-registry` | 
| ECR_WEB_REPOSITORY | `inventory-web-tier-registry` | 
| LOGIC_CONTAINER_NAME | `inventory-logic-container` | 
| LOGIC_ECS_CLUSTER | `inventory-cluster` | 
| LOGIC_ECS_SERVICE | `inventory-logic-service` | 
| LOGIC_TASK_DEFINITION_FAMILTY | `inventory-logic-task-definition` | 
| WEB_CONTAINER_NAME | `inventory-web-container` | 
| WEB_ECS_CLUSTER | `inventory-cluster` | 
| WEB_ECS_SERVICE | `inventory-web-service` | 
| WEB_TASK_DEFINITION_FAMILTY | `inventory-web-task-definition` | 

### Workflows

- There are 2 workflow files:
  - [`web-ci-cd.yml`](.github/workflows/web-ci-cd.yml) - Builds and deploys the web tier when changes are detected in the `/web-tier` directory.
  - [`logic-ci-cd.yml`](.github/workflows/logic-ci-cd.yml) - Builds and deploys the logic tier when changes are detected in the `/logic-tier` directory.
- Each workflow involves
  - Setting up the environment.
  - Build and test the application on code push.
  - Authenticate with AWS.
  - Package the application as a Docker image and push to Amazon ECR.
  - Update ECS task definition with new image.
  - Deploy updated task definition to ECS.

Now we are all done. Trigger the workflows by pushing the code to github repository.

## Screenshots from AWS console

### ECR console
<img width="1919" height="877" alt="inventory-ecr-registries" src="https://github.com/user-attachments/assets/4f1e3428-0740-4ac4-be36-fb7ae9aabb91" />

### VPC resource map
<img width="1846" height="492" alt="vpc-resource-map" src="https://github.com/user-attachments/assets/57f27cb8-4750-4c98-a21c-8180b5ad8ce3" />

### Security Groups
<img width="1919" height="865" alt="inventory_sgs" src="https://github.com/user-attachments/assets/98142cc2-c1b1-41a9-9fe9-943871ba4381" />

### Web ECS service - health
<img width="1919" height="864" alt="inventory-web-service-health" src="https://github.com/user-attachments/assets/aeff7c71-a0da-416c-b198-5e28054f0b9f" />

### Web ECS service - tasks
<img width="1918" height="868" alt="inventory-web-service-tasks" src="https://github.com/user-attachments/assets/280d9911-cc9a-4bab-a721-9d98c51880c9" />

### Logic ECS service - health
<img width="1914" height="866" alt="inventory-logic-service-health" src="https://github.com/user-attachments/assets/b2f314f9-f09d-454d-bf8f-b5296d3687df" />

### Logic ECS service - tasks
<img width="1919" height="865" alt="inventory-logic-service-tasks" src="https://github.com/user-attachments/assets/2eb2633b-c9dd-487d-962f-74f9696d975c" />

### Demo

https://github.com/user-attachments/assets/39afc3d9-6566-4b9f-a203-7f873e2334f9

