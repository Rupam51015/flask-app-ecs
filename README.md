# Flask App — AWS ECS Deployment

A minimal Flask web application built for learning containerization and deployment to **AWS ECS (Elastic Container Service)**.

![Python](https://img.shields.io/badge/Python-3.14-blue)
![Flask](https://img.shields.io/badge/Flask-3.1.1-green)
![Docker](https://img.shields.io/badge/Docker-Ready-2496ED)
![AWS ECS](https://img.shields.io/badge/AWS-ECS-FF9900)

## Features

- Responsive landing page with modern glassmorphism UI
- `/health` endpoint for ECS load balancer health checks
- Two Dockerfiles — simple and multistage (`Dockerfile-multi`)

## Tech Stack

| Component | Technology |
|-----------|------------|
| Framework | Flask 3.1.1 |
| Runtime   | Python 3.14 |
| Container | Docker (python-slim) |
| CI/CD     | Jenkins (Docker-in-Docker Setup) |
| Webhook   | Smee.io Webhook Proxy |
| Deploy    | AWS ECS |

## Project Structure

```
flask-app-ecs/
├── app.py                 # Flask app with routes
├── run.py                 # Entry point (host 0.0.0.0, port 80)
├── requirements.txt       # Python dependencies
├── templates/
│   └── index.html         # Landing page
├── Dockerfile             # Simple single-stage build
└── Dockerfile-multi       # Multistage build with distroless
└── docker-compose.yaml    # Local orchestration for testing & verification
└── Jenkinsfile            # Jenkins CI/CD pipeline script
```

## Quick Start

### Run locally

```bash
pip install -r requirements.txt
python run.py
```

App runs at **http://localhost:8000**.

### Run with Docker

**Simple build:**

```bash
docker build -t flask-app .
docker run -p 80:80 flask-app
```

**Multistage build (smaller, production-grade):**

```bash
docker build -f Dockerfile-multi -t flask-app .
docker run -p 80:80 flask-app
```

---

## CI/CD Pipeline Setup (Jenkins in Docker)
This project uses a nested **Docker-in-Docker (DinD)** architecture to run an isolated Jenkins automation server alongside an independent Docker daemon container. This approach ensures that Jenkins pipelines can build images, spin up Compose stacks, and run isolated system integration/health check tests safely without affecting your host machine's global environment.

### 1. Launch the Isolated Docker Daemon (DinD)
To route application traffic through the container engine to your local Mac browser, you must explicitly expose port `8000` via a dedicated network tunnel hook:

```bash
docker run \
  --name jenkins-docker \
  --rm \
  --detach \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  --publish 8000:8000 \
  docker:dind \
  --storage-driver overlay2
```
### 2. Launch the Jenkins Web Automation Server
Start your custom Jenkins controller container configured to talk to the sidecar DinD service securely:

```bash
docker run \
  --name jenkins-blueocean \
  --restart=on-failure \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.568.2-1
```
*   Access the Jenkins dashboard panel locally via **http://localhost:8080**

---

## Automated Webhook Triggers (Smee.io Proxy)

Because your Jenkins server runs locally inside an isolated Docker container on your Mac, it cannot accept direct inbound calls from external GitHub servers. We use **Smee.io** as a payload delivery proxy tunnel to safely bypass local network firewall barriers and loopback restrictions without modifying internal HTTP headers.

### 1. Configure the GitHub Webhook Channel
1. Visit **[smee.io](https://smee.io)** and initialize a new channel payload space. 
2. Copy your generated unique endpoint (e.g., `https://smee.ioWKnrOxH0zpTe24NE`).
3. Navigate to your repository page on GitHub ➡️ **Settings** ➡️ **Webhooks** ➡️ **Add Webhook**.
4. Configure these exact mapping parameters:
   * **Payload URL:** `https://smee.ioWKnrOxH0zpTe24NE` (Paste your unique channel URL)
   * **Content type:** `application/json`
   * **Trigger actions:** `Just the push event`
5. Click **Add Webhook**.

### 2. Run the Smee Client Local Forwarder Proxy
Execute this clean, memory-safe ephemeral terminal forwarder command on your Mac host machine (outside Docker) to bridge Smee's web network stream straight to your pipeline's background plugin router:

```bash
npx smee-client --url https://smee.ioWKnrOxH0zpTe24NE --path /github-webhook/ --port 8080
```
*   Keep this terminal process window active during development. It forwards webhooks to `http://127.0.0`.

### 3. Activate the Pipeline Trigger in Jenkins
1. Open your pipeline configuration screen inside your **Jenkins UI dashboard**.
2. Scroll to the **Build Triggers** section block.
3. Check the box labeled **GitHub hook trigger for GITScm polling**.
4. Click **Save**.

Now, any code push to your GitHub repository will automatically trigger a new verification build pipeline.

---
## Endpoints

| Route     | Method | Description                     |
|-----------|--------|---------------------------------|
| `/`       | GET    | Landing page                    |
| `/health` | GET    | Health check (returns `Server is up and running`) |

## Deploy to AWS ECS

High-level steps to deploy this app on ECS:

1. **Push image to ECR**
   ```bash
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
   docker tag flask-app:latest <account-id>.dkr.ecr.<region>.amazonaws.com/flask-app:latest
   docker push <account-id>.dkr.ecr.<region>.amazonaws.com/flask-app:latest
   ```

2. **Create ECS Task Definition** — specify the ECR image, port 80, memory/CPU limits

3. **Create ECS Service** — attach to a cluster, configure desired count, link to a load balancer

4. **Configure ALB** — target group pointing to port 80, use `/health` as the health check path
