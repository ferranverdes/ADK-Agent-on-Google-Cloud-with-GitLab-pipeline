# ADK Agent on Google Cloud with GitLab Pipeline

## ⚡ TL;DR

* Production-ready **conversational AI agent** (`Google ADK`) themed as a Barcelona tour guide, served via a minimal Python web app.  
* Backed by an **Ollama GPU inference backend** (e.g. `mistral:7b`) running on Google Cloud Run with L4 GPUs.  
* Fully automated **build → deploy** pipeline on GitLab CI using **Pulumi (Python) + GCP**.  
* Single environment wired to the current Pulumi stack (for now configured as `production` in `Pulumi.main.yaml`).  
* Infrastructure as Code creates:
  * **Artifact Registry** repository for container images.  
  * **Cloud Run** services for both the Ollama backend and the ADK agent frontend.  
  * Minimal IAM & API enablement (Cloud Run, IAM, Compute Engine, Resource Manager, Artifact Registry).  
* GitLab pipeline uses **Docker-in-Docker** plus Pulumi to:
  * Build and push the **Ollama backend image** and the **ADK agent image**.  
  * Deploy both services to Cloud Run and export their URLs as stack outputs.  

This project is focused on demonstrating how to deploy a GPU-backed LLM agent to Google Cloud using **Pulumi + GitLab CI**, not on complex business logic.

## 🧩 Project Context

This repository contains everything needed to:

* Build a **GPU-accelerated Ollama backend** on Google Cloud Run.  
* Build and serve a **Google ADK agent** (named "Ona") that answers as a Barcelona local guide.  
* Orchestrate builds and deployments using **Pulumi (Python)** from a **GitLab CI pipeline**.  

There is no traditional REST API with multiple endpoints; the core value is the **agent + model** running in the cloud and reachable via Cloud Run.

### Main components

* `ollama-backend/` → Dockerized Ollama server with GPU-friendly settings and pre-pulled `mistral:7b` model.  
* `adk-agent/` → Python app that defines the production ADK agent and runs it as a web service (`adk web`).  
* `environments/` → Pulumi stacks (build & deploy), shared modules, and GCP configuration.  
* `.gitlab-ci.yml` → GitLab pipeline that:
  * Builds and pushes container images to Artifact Registry.  
  * Deploys both services to Cloud Run using Pulumi.  

## 🤖 The Agent & Ollama Backend

### ADK Agent (`adk-agent/production_agent/agent.py`)

The project defines a single main agent:

* Built with **Google ADK** using the `LiteLlm` model wrapper.  
* Defaults to `mistral:7b` and connects to the Ollama backend via `OLLAMA_API_BASE`.  
* Runs as a web app using the `adk web` command (see `adk-agent/Dockerfile`).  
* Persona:
  * Name: **Ona**.  
  * Role: friendly local tour guide from Barcelona.  
  * Answers **in English**.

### Ollama Backend (`ollama-backend/Dockerfile`)

The Ollama service:

* Uses the official `ollama/ollama` base image.  
* Binds to `0.0.0.0:8080` and keeps models in `/models`.  
* Pre-pulls the `mistral:7b` model at build time to avoid cold downloads on first request.  
* Keeps the model in memory (`OLLAMA_KEEP_ALIVE=-1`) to minimize latency.  
* Is deployed to Cloud Run with:
  * **GPU** (L4), CPU, memory, and autoscaling limits configured via Pulumi.  

## ☁️ Cloud Deployment (GitLab + Pulumi + GCP)

This section provides instructions for setting up credentials and initiating the automated CI/CD pipeline in GitLab.

### 1️⃣ Fork the repository

Fork this repository into your own GitLab workspace.  
You'll gain full control over the CI/CD pipelines, environment variables, and Pulumi configuration.

### 2️⃣ Create a Pulumi access token

Pulumi uses an access token to authenticate and manage your infrastructure state.

1. Go to your Pulumi account → **Personal access tokens**.  
2. Click **Create token** and copy the generated value.  
3. In your GitLab fork, go to **Settings → CI/CD → Variables** and add:  
   * **Key:** `PULUMI_ACCESS_TOKEN`.  
   * **Value:** (paste your Pulumi token).  
   * **Environment:** "All (default)"  
   * **Visibility:** ✅ Masked  
   * **Flags:** 🚫 **Unset the "Protect variable"** checkbox for demonstration purposes.  

This allows Pulumi to manage infrastructure for all environments securely during CI/CD runs.

### 3️⃣ Create the Google Cloud project

You need a Google Cloud project where the Ollama backend, Artifact Registry, and Cloud Run services will run.

Create it in **Google Cloud Console → Manage Resources → Create Project** and note the **project ID**; you will use it in the Pulumi configuration.

### 4️⃣ Create a service account for the project

The project needs a service account with permissions to deploy via Pulumi.

1. Go to **IAM & Admin → Service Accounts → Create service account**.  
2. Name it `gitlab-build-sa`.  
3. Assign the following role:  
   * `Owner` (for demonstration purposes only — in real-world scenarios, grant the minimum roles required).  
4. Generate a **JSON key** and download it to your local machine.

Pulumi code in this repository will automatically enable all required GCP APIs (Artifact Registry, Cloud Run, IAM, Compute Engine, Cloud Resource Manager) when you run the stacks, so **you do not need to manually enable any APIs on the project**.

### 5️⃣ Store credentials in GitLab (Base64 encoded)

In your GitLab project:

1. Base64-encode the service account JSON key:

   ```bash
   base64 ~/Downloads/adk-agent-project-credentials.json
   ```

2. Go to **Settings → CI/CD → Variables** and add:

   * **Key:** `GOOGLE_CREDENTIALS_B64`  
   * **Value:** the Base64-encoded JSON  
   * **Environment:** "All (default)" or scoped to the branch you use  
   * **Visibility:** ✅ Masked  

These variables allow GitLab jobs to authenticate against GCP and Pulumi.

### 6️⃣ Update Pulumi configuration files

This project uses a single logical environment, configured via `Pulumi.main.yaml` in both stacks:

```text
environments/build/Pulumi.main.yaml
environments/deploy/Pulumi.main.yaml
```

Update these files with your **project's ID** and preferred region:

* `gcp:project` → **your Google Cloud project ID**.  
* `gcp:region` → region for Artifact Registry and Cloud Run (e.g. `europe-west1`).  
* `env` → logical environment label (for demonstration purposes use `production`).  

## 🏗️ Infrastructure as Code Layout (`environments/`)

The `environments/` folder contains everything Pulumi needs:

* `requirements.txt` → Python dependencies for Pulumi (Pulumi core, GCP, Docker provider).  
* `build/__main__.py` → **Build stack**:
  * Reads `env`, `project`, `region` from Pulumi config.  
  * Loads the service account key from `environments/credentials/service-account-key.json`.  
  * Enables Compute Engine and Resource Manager APIs.  
  * Creates an **Artifact Registry** repository (`adk-agent-repo`).  
  * Builds & pushes:
    * `ollama-backend` image.  
    * `adk-agent` image.  
  * Exports image names and digests as Pulumi stack outputs.  
* `deploy/__main__.py` → **Deploy stack**:
  * Reads the same config (`env`, `project`, `region, zone`).  
  * Reads image names/digests from environment variables (injected from the GitLab build job).  
  * Loads the same service account JSON as a Pulumi provider.  
  * Deploys **two Cloud Run services** via `modules/cloud_run.py`:
    * A GPU-backed Ollama backend (with CPU/memory/GPU counts).  
    * The ADK agent service, with `OLLAMA_API_BASE` pointing to the Ollama URL.  
  * Exports `ollama_backend_url` and `agent_url` as outputs.  
* `modules/`:
  * `artifact_registry.py` → Creates an Artifact Registry repo and builds/pushes Docker images.  
  * `cloud_run.py` → Generic **public Cloud Run v2** deployment helper with optional GPU support.  
  * `resource_manager.py` & `compute_engine.py` → Ensure required APIs are enabled.  

## 🧪 GitLab CI Pipeline (`.gitlab-ci.yml`)

The pipeline defines two stages:

* **`build`** → Build container images and publish them to Artifact Registry with Pulumi.  
* **`deploy`** → Deploy Cloud Run services using those images.  

### Build stage

Key characteristics:

* Image: `pulumi/pulumi-python`.  
* Uses `docker:...-dind` as a service for Docker builds.  
* Steps:
  * Decode `GOOGLE_CREDENTIALS_B64` into `environments/credentials/service-account-key.json`.  
  * Install Pulumi Python dependencies from `environments/requirements.txt`.  
  * Run `pulumi up` in `environments/build` for stack = `$CI_COMMIT_BRANCH` (auto-creates stack if missing).  
  * Export image details as `build.env` for downstream jobs:
    * `OLLAMA_IMAGE_NAME`, `OLLAMA_IMAGE_DIGEST`  
    * `AGENT_IMAGE_NAME`, `AGENT_IMAGE_DIGEST`  
* `build.env` is exposed as a **dotenv artifact**, so the deploy job gets these variables automatically.

### Deploy stage

Key characteristics:

* Also uses `pulumi/pulumi-python`.  
* Declares `needs: [build]` to consume artifacts and run as soon as the build finishes.  
* Steps:
  * Decode `GOOGLE_CREDENTIALS_B64` into `environments/credentials/service-account-key.json`.  
  * Install Pulumi dependencies again (using cache).  
  * Run `pulumi up` in `environments/deploy` for stack = `$CI_COMMIT_BRANCH`.  
  * Export environment info as `deploy.env` (also as a dotenv report):
    * `GCP_PROJECT`, `GCP_REGION`  
    * `OLLAMA_BACKEND_URL`, `AGENT_URL`  

This gives you a fully automated path from commit → images → deployed Cloud Run URLs.

After the **deploy** phase finishes, Pulumi prints the public URLs for both services in its outputs. For example:

```bash
Updating.........
 +  gcp:cloudrunv2:ServiceIamMember adk-agent-public-invoker created (5s) 
 +  pulumi:pulumi:Stack adk-agent-infra-deploy-main created (199s) 
Outputs:
    agent_url         : "https://adk-agent-service-5i5qcxvyxq-ew.a.run.app"
    env               : "production"
    ollama_backend_url: "https://ollama-backend-service-5i5qcxvyxq-ew.a.run.app"
    project           : "adk-agent-329873"
    region            : "europe-west1"
    zone              : "europe-west1-b"
Resources:
    + 14 created
Duration: 3m21s
```

You can use the value of `agent_url` to access the deployed ADK agent in your browser:

![ADK Agent Interface][1]

## 💻 Local Development (Conceptual)

This repo is oriented around **cloud deployment** rather than local-only dev, but you can still:

* Build and run `ollama-backend` locally with Docker to test the model server.  
* Run the `adk-agent` image locally and point it at a local or remote Ollama backend.  
* Use the same Dockerfiles that the pipeline uses to match production behaviour.

Example (run Ollama backend locally):

```bash
cd ollama-backend
docker build -t local-ollama-backend .
docker run -p 8080:8080 local-ollama-backend
```

Then build and run the agent:

```bash
cd adk-agent
docker build -t local-adk-agent .
docker run -p 8081:8080 \
  -e OLLAMA_API_BASE="http://host.docker.internal:8080" \
  local-adk-agent
```

Visit `http://localhost:8081` (or the endpoint exposed by `adk web`) to interact with the agent.

## 🧩 Tech Stack Overview

| Layer        | Technology                  | Purpose                                      |
|-------------|-----------------------------|----------------------------------------------|
| Agent       | Google ADK + LiteLlm        | Defines and serves the conversational agent  |
| Model       | Ollama (`mistral:7b`)       | LLM inference backend                        |
| Container   | Docker                      | Packaging for Cloud Run                      |
| IaC         | Pulumi (Python)             | Infrastructure automation on GCP             |
| CI/CD       | GitLab CI                   | Build & deploy pipeline                      |
| Platform    | Google Cloud Run (GPU + CPU)| Serverless hosting for backend & agent       |
| Registry    | Artifact Registry           | Stores container images                      |

## 🎯 Learning Objectives

This project demonstrates how to:

* Package a **GPU-backed Ollama LLM** and an **ADK agent** as Dockerized services.  
* Use **Pulumi (Python)** to define GCP infrastructure (Artifact Registry + Cloud Run + IAM + APIs).  
* Wire **GitLab CI** to run Pulumi stacks in separate `build` and `deploy` phases.  
* Pass image metadata between jobs using **dotenv artifacts**, enabling reproducible deployments.  

[1]: https://gitlab.com/ferran.verdes/static/-/raw/main/images/adk-agent-on-google-cloud-with-gitlab-pipeline-adk-interface.png
