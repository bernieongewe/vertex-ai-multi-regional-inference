# Multi-Regional Vertex AI Inference with Global Load Balancing

This repository contains the code and configuration steps to build a resilient, multi-regional inference architecture on Google Cloud Platform. 

This MLOps pattern is designed to overcome **regional capacity constraints** (such as GPU stockouts) and ensure high availability for mission-critical models. By using a "Smart Router" pattern with Cloud Run and Private Service Connect, we can achieve absolute failover between regions behind a single Global Load Balancer IP.

### 📚 Read the Blog Post
For a deep dive into the architecture, the motivation behind the "Smart Router" pattern, and the specific networking challenges this solution solves, please read the accompanying blog post:

👉 **[Multi-Regional Inference with Vertex AI](https://medium.com/@o.bernie/multi-regional-inference-with-vertex-ai-9c750fc7c9c3)**

---

## ⚠️ Important: Configuration Variables

**Please read this before running any code.**

These notebooks rely heavily on environment variables (`os.environ`) to configure Project IDs, Region names, and Resource IDs. 

* **Do not blindly run the cells.** You must update the variables in the first cell of each notebook to match your specific GCP environment.
* **Project IDs:** Pay close attention to `CONSUMER_PROJECT_ID` (where the Load Balancer/Cloud Run lives) vs `PRODUCER_PROJECT_ID` (where the Vertex Endpoints live). In many setups, these are the same project, but ensure they are correct.
* **Endpoint IDs:** Notebook `03` requires the specific numeric Endpoint IDs generated in Notebook `02`. You must copy-paste these correctly for the router to work.

---

## Repository Structure

The workflow is divided into three sequential notebooks. You must run them in order.

### 1. `01-create-clean-model.ipynb`: Model Training & Registration
This notebook handles the machine learning artifacts.
* Trains a simple TensorFlow model (Boston Housing prices).
* Exports the model in the correct `SavedModel` format required by Vertex AI.
* Uploads the artifacts to Google Cloud Storage (GCS).
* Registers the model in the **Vertex AI Model Registry**.

### 2. `02-deploy-clean-model.ipynb`: Regional Deployment
This notebook handles the initial Vertex AI setup.
* Creates **Vertex AI Endpoints** in two separate regions (e.g., `us-central1` and `us-east4`).
* Deploys the model from the registry to both endpoints.
* *Note:* This step creates the "standard" regional endpoints that we will later bridge together.

### 3. `03-build-lb-and-cloud-run.router.ipynb`: Global Networking & Smart Router
This is the core infrastructure notebook. It builds the failover architecture.
* **Private Service Connect (PSC):** Creates internal IPs and Forwarding Rules with **Global Access** enabled, allowing cross-region connectivity.
* **Smart Router (Cloud Run):** Generates and deploys a Python/Flask proxy that handles authentication, protocol translation (HTTP -> HTTPS), and URL rewriting.
* **Global Load Balancer:** Configures a Global External Application Load Balancer with Serverless NEGs to route traffic to the Cloud Run instances.
* **Verification:** Includes scripts to test "Absolute Failover" logic.

---

## Prerequisites

* A Google Cloud Project with billing enabled.
* A Python environment (Vertex AI Workbench is recommended).
* The following APIs enabled:
    * Vertex AI API
    * Compute Engine API
    * Cloud Run API
    * Artifact Registry API
* Permissions: `Editor` role or equivalent permissions for Network, Compute, and AI Platform.

## Usage

1.  Clone this repository into your Vertex AI Workbench instance.
2.  Open `01-create-clean-model.ipynb`, update the user configuration variables, and run all cells.
3.  Open `02-deploy-clean-model.ipynb`, update the variables, and run all cells. **Note the Endpoint IDs generated at the end.**
4.  Open `03-build-lb-and-cloud-run.router.ipynb`, paste the Endpoint IDs from step 2 into the configuration cell, and run the networking setup.

## Running Predictions

Once deployed, you cannot use the standard Vertex AI SDK to query the Load Balancer (as the SDK constructs regional URLs). You must use `curl` or `requests` to hit the Global LB IP:

```bash
curl -k -X POST "https://<LOAD_BALANCER_IP>/predict" \
    -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
    -H "Content-Type: application/json" \
    -d '{ "instances": [[...]] }'
```

> **Note:** This is a personal project and is not an officially supported Google solution.
