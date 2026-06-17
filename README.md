# Deploy API

Docker and AWS ECS Fargate deployment

## Local Development

```bash
pip install -r requirements.txt
uvicorn main:app --reload
```

## Docker

```bash
# Build
docker build -t deploy-api .

# Run
docker run -p 8000:8000 deploy-api

# Run with env vars
docker run --env-file .env -p 8000:8000 deploy-api
```

## Docker Compose

```bash
docker compose up
```

## AWS Setup

### 1. ECR (Image Registry)
- Create a repository named `testing-deploy`
- Images are pushed here by the GitHub Actions workflow

### 2. IAM
- Create an IAM user with `ecr:*` and `ecs:UpdateService` permissions
- Add access keys to GitHub Secrets:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`
- Create a `taskExecutionRole` with:
  - `AmazonECSTaskExecutionRolePolicy`
  - `secretsmanager:GetSecretValue` (if using Secrets Manager)

### 3. Secrets Manager (optional)
- Store all env vars as key/value pairs in one secret
- Reference each key in the Task Definition using the ARN:
  ```
  arn:aws:secretsmanager:us-east-2:ACCOUNT_ID:secret:SECRET_NAME:KEY_NAME::
  ```

### 4. Task Definition
- Launch type: Fargate
- Architecture: ARM64
- Container settings:
  - Image URI: `ACCOUNT_ID.dkr.ecr.us-east-2.amazonaws.com/testing-deploy:latest`
  - Port: 8000
  - Logging: awslogs with `awslogs-create-group: true`
- For the worker, override the command with: `python3,worker.py`

### 5. Cluster
- Create an ECS cluster (Fargate handles the infrastructure)

### 6. Service
- Create one service per container (API, worker)
- Attach a Security Group with inbound TCP port 8000 open to `0.0.0.0/0`
- Enable Auto-assign public IP (for testing)

### 7. Load Balancer (production)
- Create an Application Load Balancer (ALB)
- Create a Target Group pointing to the ECS service on port 8000
- Attach the ALB to the ECS service for a stable public URL

### 8. Custom Domain (optional)
- Create an A record in Route 53 pointing to the ALB
- Add an SSL certificate via ACM
- Add an HTTPS listener (port 443) to the ALB

## CI/CD

Pushing to `main` automatically:
1. Builds the Docker image (ARM64)
2. Pushes it to ECR with the `latest` tag
3. Triggers a new ECS deployment

## Project Structure

```
.
├── main.py           # FastAPI application
├── worker.py         # Background worker
├── requirements.txt  # Python dependencies
├── Dockerfile        # Docker image definition
├── docker-compose.yml
└── .github/
    └── workflows/
        └── deploy.yml  # GitHub Actions CI/CD
```
