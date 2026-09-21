# Vercel-Like Deployment Platform

A self-hosted deployment platform inspired by **Vercel**, built to understand how modern deployment platforms handle source code, build processes, job queues, object storage, and static asset delivery.

The platform allows a user to provide a **GitHub repository URL**, automatically clone the project, upload its source files to object storage, queue a deployment job, build the project, and store the resulting static assets for serving.

## 🚀 How It Works

The platform is divided into three major services:

```text
                    ┌──────────────────┐
                    │     Frontend     │
                    │  GitHub Repo URL │
                    └────────┬─────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │   Upload Service    │
                  │      Express        │
                  └─────────┬───────────┘
                            │
                     Clone Repository
                            │
                            ▼
                  ┌─────────────────────┐
                  │   Object Storage    │
                  │      S3 / R2        │
                  └─────────┬───────────┘
                            │
                     Deployment Job
                            │
                            ▼
                  ┌─────────────────────┐
                  │    Redis Queue      │
                  │       build-Q       │
                  └─────────┬───────────┘
                            │
                            ▼
                  ┌─────────────────────┐
                  │ Deployment Service  │
                  │     Build Worker    │
                  └─────────┬───────────┘
                            │
                   npm install
                   npm run build
                            │
                            ▼
                  ┌─────────────────────┐
                  │   Built Artifacts   │
                  │      S3 / R2        │
                  └─────────┬───────────┘
                            │
                            ▼
                  ┌─────────────────────┐
                  │  Request Handler    │
                  │ Static File Server  │
                  └─────────────────────┘
```

The architecture separates **uploading**, **building**, and **serving** so that CPU-intensive build workloads do not block lightweight upload operations.

---

## ✨ Features

* Deploy projects directly from a GitHub repository
* Automatically clone repositories
* Recursively discover project files
* Upload source files to S3 / Cloudflare R2
* Generate unique deployment IDs
* Queue deployment jobs using Redis
* Process builds asynchronously
* Run `npm install` and `npm run build`
* Upload generated build artifacts
* Serve static deployment files
* Architecture designed for future worker scaling
* TypeScript-based Node.js backend

---

## 🏗️ Architecture

### 1. Upload Service

The Upload Service is responsible for receiving a GitHub repository URL and preparing it for deployment.

Workflow:

```text
GitHub URL
    ↓
Clone Repository
    ↓
Generate Deployment ID
    ↓
Find All Files
    ↓
Upload Files → S3 / R2
    ↓
Push Deployment ID → Redis
    ↓
Return Deployment ID
```

The service uses Express for the API, Simple Git for repository cloning, and an S3-compatible object store for source files.

### 2. Deployment Service

The Deployment Service works as a background worker.

It continuously listens for deployment jobs in Redis:

```text
Redis Queue
     ↓
Get Deployment ID
     ↓
Download Source
     ↓
npm install
     ↓
npm run build
     ↓
Upload Build Output
```

Redis's blocking queue operation allows the worker to wait for new jobs without repeatedly making unnecessary requests.

### 3. Request Handler

After a project has been built, the Request Handler serves the generated static files.

```text
User Request
     ↓
Request Handler
     ↓
Object Storage
     ↓
HTML / CSS / JS
```

A caching layer can be added between the request handler and object storage to reduce repeated storage requests and improve response times.

---

## 🧰 Tech Stack

| Technology        | Purpose                                    |
| ----------------- | ------------------------------------------ |
| **Node.js**       | Backend runtime                            |
| **TypeScript**    | Type-safe development                      |
| **Express**       | HTTP API                                   |
| **Simple Git**    | GitHub repository cloning                  |
| **Redis**         | Deployment job queue                       |
| **AWS S3**        | Object storage                             |
| **Cloudflare R2** | S3-compatible storage alternative          |
| **Docker**        | Containerization / future worker isolation |
| **Postman**       | API testing                                |

The core implementation uses Node.js/TypeScript, Express, Simple Git, AWS-compatible object storage, and Redis queues.

---

## 📂 Deployment Flow

A complete deployment follows these steps:

### Step 1 — Submit Repository

The client sends a GitHub repository URL to the deployment API.

```http
POST /deploy
```

### Step 2 — Clone Repository

The Upload Service clones the repository into a temporary deployment directory.

### Step 3 — Upload Source

All files are recursively discovered and uploaded to object storage under a unique deployment ID.

```text
output/<deployment-id>/
```

### Step 4 — Queue Deployment

The deployment ID is pushed into the Redis `build-Q` queue.

### Step 5 — Build

The Deployment Service retrieves the job and downloads the source files.

It then executes:

```bash
npm install
npm run build
```

### Step 6 — Upload Build Artifacts

The generated static files are uploaded back to object storage.

### Step 7 — Serve

The Request Handler serves the generated HTML, CSS, JavaScript, and other static assets.

The complete workflow follows the eight-step pipeline described in the project architecture.

---

## 🔑 Core Functions

| Function             | Responsibility                  |
| -------------------- | ------------------------------- |
| `generate()`         | Creates deployment IDs          |
| `cloneRepo()`        | Clones GitHub repositories      |
| `getAllFiles()`      | Recursively finds project files |
| `uploadFile()`       | Uploads an individual file      |
| `uploadAllFiles()`   | Uploads project files           |
| `pushToQueue()`      | Adds deployment jobs to Redis   |
| `deployLoop()`       | Processes deployment jobs       |
| `downloadS3Folder()` | Downloads source files          |
| `buildProject()`     | Runs the project build          |

These functions form the core upload and deployment pipeline.

---

## ☁️ Storage

The platform uses an **S3-compatible object storage architecture**.

Possible providers include:

* AWS S3
* Cloudflare R2

Each deployment is isolated using a deployment-specific object prefix:

```text
output/
├── deployment-001/
│   ├── index.html
│   ├── assets/
│   └── ...
│
├── deployment-002/
│   ├── index.html
│   ├── assets/
│   └── ...
```

Cloudflare R2 can be used as an alternative to AWS S3 while maintaining compatibility with the S3 API model.

---

## 🔐 Environment Variables

Create a `.env` file in the backend services.

```env
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_REGION=your_region
AWS_BUCKET_NAME=your_bucket

REDIS_URL=your_redis_connection_url
```

> Never commit `.env` files or cloud credentials to GitHub.

---

## 🛠️ Running Locally

### Clone the repository

```bash
git clone https://github.com/your-username/your-repository.git

cd your-repository
```

### Install dependencies

```bash
npm install
```

### Start the Upload Service

```bash
npm run dev
```

### Start the Deployment Worker

Run the deployment worker separately:

```bash
npm run worker
```

The worker continuously listens for deployment jobs from Redis.

---

## 🧪 Testing

The API can be tested using Postman or any HTTP client.

Example:

```http
POST /deploy
Content-Type: application/json
```

```json
{
  "repoUrl": "https://github.com/user/project"
}
```

The server returns a deployment ID that can be used to track the deployment.

---

## 📈 Scalability

The architecture intentionally separates the services so they can scale independently.

For example:

```text
                  Redis Queue
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      Worker 1    Worker 2    Worker 3
          │           │           │
        Build       Build       Build
```

More deployment workers can be introduced as the number of builds increases.

For production-scale infrastructure, containerized workers and orchestration platforms such as Kubernetes or managed container services can be introduced.

---

## 🎯 What This Project Demonstrates

This project is primarily an exploration of **deployment infrastructure and distributed backend architecture**.

It demonstrates:

* Asynchronous job processing
* Redis-based queues
* Object storage
* Git repository automation
* Build pipelines
* Service separation
* Static asset delivery
* Cloud infrastructure concepts
* Containerization concepts
* Scalable worker architecture

---

## 🚧 Future Improvements

Potential improvements include:

* Docker-based isolated build environments
* Multiple concurrent deployment workers
* Deployment status tracking
* Build logs
* Automatic GitHub webhook deployments
* Custom domains
* HTTPS certificates
* CDN integration
* Deployment history
* Rollbacks
* Build caching
* Authentication
* Resource limits for builds
* Production-grade sandboxing

---

## 📚 Project Inspiration

This project was built as a learning exercise to understand the architecture behind modern deployment platforms such as Vercel, particularly the separation between source uploading, asynchronous builds, object storage, and static asset serving.

---

## 👨‍💻 Author

**Shahid Sekh**

BCA Student | Full-Stack Developer | Backend & Cloud Enthusiast

---

## ⭐ If You Found This Useful

If this project helped you understand deployment infrastructure, consider giving the repository a ⭐.
