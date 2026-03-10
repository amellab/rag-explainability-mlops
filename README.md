# RAG Explainability and MLOps Pipeline
<img width="987" height="378" alt="Image" src="https://github.com/user-attachments/assets/2833764d-3957-4270-bac1-30cd6a100bb9" />

## 1. Overview

This project implements a distributed production grade `RAG (Retrieval Augmented Generation) pipeline` on AWS.  
The goal is to migrate away from a slow monolithic Lambda setup and build a scalable low latency inference workflow using managed cloud services.  


The final system achieves:

- **~35 percent improvement** in model inference latency by migrating FLAN generation to SageMaker Realtime endpoints
- **~80 percent reduction** in end to end API latency after replacing Serverless inference with low latency realtime serving
- A fully distributed RAG pipeline where embedding, retrieval, and generation run as independent scalable services

This project demonstrates practical ML serving design with AWS and Pinecone.

## 2. Tech Stack

<p align="left">
   <img src="https://custom-icon-badges.demolab.com/badge/AWS-%23FF9900.svg?logo=aws&logoColor=white" alt="AWS Badge" width="80">
   <img src="https://img.shields.io/badge/Docker-2496ED.svg?logo=docker&logoColor=white" alt="Docker Badge" width="100">
   <img src="https://img.shields.io/badge/GitHub_Actions-2088FF.svg?logo=githubactions&logoColor=white" alt="GitHub Actions Badge" width="160">
   <img src="https://img.shields.io/badge/Streamlit-FF4B4B.svg?logo=streamlit&logoColor=white" alt="Streamlit Badge" width="115">
   <img src="https://img.shields.io/badge/Pinecone-3776E0.svg?logo=data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMjQiIGhlaWdodD0iMjQiIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj4KPHBhdGggZD0iTTEyIDIyQzYuNDc3MTQgMjIgMiAxNy41MjI5IDIuMTM4MjQgMTJDMi4yNTM1IDcuMjk3MzggNi40OTU4NCAzLjA1NTE0IDExLjIwMjQgMy4wMzkyOEMxNi4yMDI0IDMuMDI1NDMgMjAuMTc2MiA3LjAyNTQzIDIwLjAzOTUgMTJDMjAuMDExMyAxNy41MTg0IDE1LjUxODQgMjIgMTIgMjJaTTExLjk5NjkgNC41QzcuOTg4NjYgNC41IDQuNSA3Ljk4ODY2IDQuNSAxMi4wMDMzQzQuNSAxNi4xMTUzIDcuODc5NTEgMTkuNDkzNiAxMi4wMDMzIDE5LjQ5MzZDMTYuMDExNiAxOS40OTM2IDE5LjUgMTYuMDE2OSAxOS41IDEyLjAwMzNDMTkuNS03Ljk4ODY2IDE2LjAxMTYgNC41IDExLjk5NjkgNC41WiIgZmlsbD0id2hpdGUiLz4KPC9zdmc+" width="113">


   
</p>


| Category        | Tools & Services                                 |
|-----------------|---------------------------------------------------|
| **Compute**     | AWS Lambda, SageMaker Realtime, ECS Fargate       |
| **Storage**     | Amazon S3, Amazon ECR                             |
| **Retrieval**   | Pinecone, E5 Embeddings                           |
| **Orchestration** | API Gateway, AWS Lambda                         |
| **Observability** | CloudWatch Logs, CloudWatch Metrics             |
| **CI/CD**       | GitHub Actions → ECS Fargate                      |
| **UI**          | Streamlit (Docker container)                      |


## 3. Key Features

- **Decoupled ML services**  
  Embedding, retrieval, and generation run as isolated services, allowing independent scaling and fault isolation.

- **Production grade orchestration**  
  Lambda orchestrates cross service calls with request tracing, timeouts, and error handling for reliable inference.

- **Realtime inference with consistent latency**  
  SageMaker Realtime endpoints ensure predictable response times with no cold starts, improving downstream UX.

- **Efficient vector search pipeline**  
  Pinecone retrieval is integrated with E5 embeddings to deliver high recall and fast top K similarity search.

- **Cloud native observability**  
  Custom CloudWatch metrics track latency, retrieval quality, endpoint performance, and error behavior across the pipeline.

## 4. Architecture
<img width="1123" height="171" alt="Image" src="https://github.com/user-attachments/assets/4aaa071e-4aa0-4b8d-b6b2-66740d68f681" />

The system implements a fully distributed multi stage RAG pipeline.  
Each stage—embedding, retrieval, and generation—is deployed as an independent managed service behind AWS endpoints.

### 4-1. System Components

- **AWS API Gateway** → public entry point for the UI  
- **AWS Lambda (Orchestrator)** → routes the request across embedding, retrieval, and generation services  
- **Amazon SageMaker Realtime Endpoint (E5)** → embedding model that converts the query into vector representations  
- **Pinecone** → vector retriever performing top K similarity search  
- **Amazon SageMaker Realtime Endpoint (FLAN)** → generator LLM producing the final answer  
- **AWS Lambda (Response Handler)** → assembles the final response and returns it to API Gateway  

### 4-2. Architecture transition

- **Before:**  
  - A single monolithic Lambda loaded E5 + FLAN models inside the function  
  - Large package size, long cold starts  
  - Highly unstable and slow latency across the entire API

- **After:**  
  - Fully distributed RAG pipeline with independent ML services  
  - Zero cold starts due to SageMaker Realtime  
  - Stable and predictable end to end latency  
  - Easy horizontal scaling at each stage (embedder, retriever, generator)
 
### 4-3. S3 Usage in the Pipeline

- Serves as the central model and data store for production inference
- Stores the E5 embedding model artifact (`e5-large-model.tar.gz`) for the embedding SageMaker endpoint  
- Stores the FLAN generation model artifact (`flan-t5-small.tar.gz`) for the generation SageMaker endpoint  

## 5. Observability Setup 

To ensure accurate latency measurement, the system includes lightweight observability:

- **CloudWatch Logs** for Lambda and ECS (inference logs, errors, request traces)
- **CloudWatch Metrics** for ModelLatency, ErrorCount, and end-to-end request duration
- A custom metric emitter inside the client script (`Latency`, `ErrorCount`) for precise timing

All performance numbers below were collected from these CloudWatch dashboards.


## 6. Performance Summary

| Metric                 | Before   | After  | Improvement          |
| ---------------------- | -------- | ------ | -------------------- |
| Model latency          | 1.0–1.5s | 0.85s  | ~35% improvement    |
| API end to end latency | ~2.4s    | ~0.49s | ~80% reduction |


### 6-1. Model Inference Latency (FLAN)

- **Before (Lambda monolithic):**  
  FLAN weights were loaded inside the Lambda function, causing warm latency in the **1.0–1.5s** range.  
  Cold starts (7–15s) were excluded from comparison because they are nondeterministic and not meaningful for percentage metrics.

- **After:**  
  Switching FLAN to a SageMaker Realtime endpoint produced a stable model-level inference latency of **~0.85s**  
  Verified through:  
  - SageMaker `ModelLatency` metric  
  - Controlled 10-request benchmark  
  - Repeated warm-path inference tests

### 6-2. API End to End Latency

- **Before:**  
  The pipeline used **Serverless FLAN**, which had inconsistent latency (3–5s spikes).  
  Measured end-to-end values from CloudWatch custom metric `RAG_MLOps.Latency` averaged **~2.4s**.

- **After:**  
  Switching FLAN to Realtime inference produced a stable API end-to-end latency of **~0.49s**  
  Verified through:
  - CloudWatch Metrics  
  - Direct UI-driven test runs  
  - API duration tracking from Lambda

## 7. Runtime Deployment Architecture (AWS ECS Fargate)

<img width="762" height="385" alt="Image" src="https://github.com/user-attachments/assets/d564f408-bda0-4e97-ad61-a7fce588e638" />

### 7-1. Role of ECS Fargate in this project

Fargate serves a **frontend layer** on top of the RAG inference pipeline:

1. Provides a publicly accessible dashboard used for real time testing  
2. Sends queries through API Gateway → Lambda → SageMaker → Pinecone  
3. Visualizes latency, outputs, and debugging logs  
4. Streams metrics to CloudWatch through the container log driver  

It does *not* run ML models — it only hosts the UI that interacts with them.

### 7-2. Benefits

- No VM management (serverless container runtime)
- Seamless auto scaling at the container level
- Built in CloudWatch logging integration
- Clean separation between frontend and backend inference services

## 8. Local Development

Local development is split into two environments depending on the purpose of the project.  
(1) Research explainability workflow  
(2) AWS based production pipeline  

### 8-1. Local Explainability Environment (Research Mode)

This environment runs the FAISS-based retrieval pipeline used in the paper.

```
conda activate rag-explain
pip install -r requirements.txt

# Launch the explainability dashboard
streamlit run dashboard/app.py
```

**This version uses:**
- **Embedder:** `intfloat/e5-large` (SentenceTransformer)
- **Retriever:** FAISS HNSW index
- **Generator:** `google/flan-t5-base` running on CPU
- **Pipeline:** Query → Retrieve → Combine → Generate
- No AWS connectivity required

### 8-2. AWS Pipeline Environment (Production Mode)

This environment connects the UI to the full AWS RAG stack:  
**`API Gateway → Lambda → SageMaker (E5) → Pinecone → SageMaker (FLAN)`**  


```
conda activate rag-explain
pip install -r mlops_dashboard/requirements.txt

# Run the production-linked dashboard
streamlit run mlops_dashboard/app.py
```

**This version uses:**
- API calls to the deployed AWS endpoint
- CloudWatch metrics emission
- ECS-compatible UI layout

## 9. CI/CD Pipeline (GitHub Actions → AWS ECS Fargate)

<img width="996" height="140" alt="Image" src="https://github.com/user-attachments/assets/ea440528-ac11-4e5a-bc7c-4db0b166cb3f" />

ECS Fargate deployment is fully automated through **GitHub Actions**.  
This pipeline builds the Docker image, pushes it to ECR, updates the ECS service, and rolls out the new UI version without manual steps.


## Supplementary: Explainability and Evaluation Resources

### S1. Research Publication Status
- **Title**: Explainability Analysis of Retrieval-Driven Behavior in RAG Pipelines
- **Status**: Published as a Preprint on **Zenodo** (February 2026)
- **Identifier[DOI]**: : 10.5281/zenodo.18945160
- **Includes**: Embedding similarity and variance analysis, retrieval stability and drift evaluation, generation robustness and length analysis, and pipeline-level explainability of RAG behavior.


### S2. Full Explainability Reports & Notebooks

All evaluation reports and experimental notebooks used in the published explainability study are available in the [docs](https://github.com/ameliekihm/rag-explainability-mlops/tree/main/docs) and [notebooks](https://github.com/ameliekihm/rag-explainability-mlops/tree/main/notebooks) directories.
