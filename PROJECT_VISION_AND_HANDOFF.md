# DOCUMENT-KNOWLEDGE-STORE

## Project description

A self-hosted, multi-tenant enterprise knowledge platform for securely uploading documents, organizing them into access-controlled topics, and asking questions through Retrieval-Augmented Generation (RAG) with source citations. The platform is deployed locally on Kubernetes using GitOps and observable microservices.

## Vision

The goal of `DOCUMENT-KNOWLEDGE-STORE` is to build a realistic, production-style distributed RAG platform without depending on cloud infrastructure or paid AI APIs.

Organizations will be able to store their documents locally, separate knowledge by tenant and topic, control which users can access each topic, and obtain grounded answers that cite the source documents used to generate the response.

The project is intentionally focused on DevOps, platform engineering, distributed systems, GitOps, service networking, storage, observability, resilience, and operational practices. Application code should remain focused and simple enough that the infrastructure and system design remain the main learning objectives.

## Core capabilities

### Administration

An administrator will be able to:

- Create and delete tenants or organizations.
- Create and delete users.
- Create and delete knowledge topics.
- Assign users to topics.
- Assign topic-level roles such as administrator, editor, and reader.
- View document ingestion status.
- Configure quotas and usage limits.

The first version will use a simple user, topic, role, and permission relationship stored in PostgreSQL. A separate custom authentication microservice is not required for the initial version. A dedicated identity provider such as Keycloak can be introduced later if authentication becomes part of the project scope.

### Document management

Authorized users will be able to:

- Upload documents to a topic.
- View documents belonging to accessible topics.
- Delete or replace documents.
- Track whether a document is uploaded, processing, ready, or failed.
- Keep original documents entirely on local infrastructure.

### RAG queries

Authorized users will be able to:

- Ask questions within a selected topic.
- Retrieve only information belonging to their tenant and permitted topic.
- Receive an answer generated from retrieved document content.
- Receive citations that identify the source document and relevant section.
- Receive an explicit response when the retrieved context is insufficient.

## High-level request flows

### Document ingestion

1. A client sends a document upload request through the Istio ingress gateway.
2. The application verifies the tenant, user, topic access, file type, and quota.
3. The raw document is stored in MinIO.
4. Document metadata and processing status are recorded in PostgreSQL.
5. An ingestion event containing document references is published to Kafka.
6. An ingestion worker reads the document from MinIO, extracts its text, and divides it into chunks.
7. The embedding component creates a vector for each chunk.
8. Vectors and searchable metadata are stored in Qdrant.
9. The final ingestion status is recorded in PostgreSQL.

The document itself must not be placed in Kafka. Kafka carries events and references; MinIO remains the source of truth for the raw file.

### Question answering

1. A client sends a question and topic identifier through the Istio ingress gateway.
2. The application validates the user's access to the requested topic.
3. The query is converted into an embedding.
4. Qdrant searches only vectors matching the tenant and topic filters.
5. The most relevant chunks are supplied to the local language model.
6. The language model generates an answer using only the supplied context.
7. The API returns the answer with citations and source-document information.

## Planned microservices

### Admin/API service

Provides external APIs for tenant, user, topic, role, permission, document, and quota operations. It is the main entry point for administrative and upload requests.

### Ingestion service

Consumes document events from Kafka, retrieves raw files from MinIO, extracts text, chunks content, requests embeddings, writes vectors to Qdrant, and updates processing status.

### RAG query service

Handles questions, verifies topic access, retrieves relevant vectors, constructs grounded prompts, calls the local LLM, and returns answers with citations.

### Model service

Provides local generation and embedding capabilities through Ollama. The application must access it through a provider interface so another local model or hosted API can be introduced later without redesigning the RAG services.

The services may begin as a small number of deployables and be separated further only when operational or scaling requirements justify it.

## Technology choices

| Component | Technology | Responsibility |
| --- | --- | --- |
| Local cluster | kind | Runs the Kubernetes control plane and worker nodes locally. |
| Package management | Helm | Installs and configures platform components. |
| GitOps | Argo CD | Reconciles the cluster with the desired state stored in Git. |
| Ingress and service mesh | Istio | Provides the local front door, traffic routing, service identity, mTLS, telemetry, and traffic policies. |
| Ingress data plane | Envoy | Proxies requests according to configuration distributed by Istio. |
| Relational database | PostgreSQL with CloudNativePG | Stores tenants, users, topics, permissions, document metadata, status, and operational relationships. |
| Object storage | MinIO | Stores original documents and derived artifacts on local persistent volumes. |
| Vector database | Qdrant | Stores chunk embeddings and performs tenant- and topic-filtered similarity search. |
| Cache and limits | Redis | Supports caching, rate limiting, temporary state, and idempotency controls. |
| Event streaming | Kafka | Decouples uploads from asynchronous ingestion and supports retries and scalable consumers. |
| Local inference | Ollama | Serves locally hosted generation and embedding models. |
| Generation model | Qwen3 14B | Generates grounded answers from retrieved document context. |
| Embedding model | Qwen3 Embedding 0.6B | Creates document and query vectors for retrieval. |
| Metrics | Prometheus | Collects Kubernetes, infrastructure, application, and service metrics. |
| Dashboards | Grafana | Displays health, performance, resource, ingestion, and RAG metrics. |
| Application language | Python | Implements the APIs, workers, ingestion logic, and RAG workflow. |

All primary components run locally. There are no required token charges, cloud storage charges, or managed-service dependencies. Operational costs are limited to the existing computer, electricity, memory, GPU usage, and disk space.

## Local AI approach

The initial model configuration is:

- `qwen3:14b` for answer generation.
- `qwen3-embedding:0.6b` for document and query embeddings.

Qwen3 14B's quantized Ollama model is suitable for the AMD Radeon RX 9070 XT with 16 GB VRAM. The smaller embedding model keeps ingestion efficient while remaining sufficient for the initial RAG workload.

Ollama may initially run directly on the host to obtain straightforward access to the GPU. Kubernetes services can call that endpoint through a controlled internal connection. Moving model serving into Kubernetes can be evaluated later; it is not required for the first working platform.

## Multi-tenancy and isolation

Every tenant-owned resource must carry a tenant identifier. Topic identifiers provide a second isolation boundary inside a tenant.

Isolation must be enforced in all relevant layers:

- PostgreSQL queries must be scoped to the tenant.
- Qdrant searches must include tenant and topic filters.
- MinIO objects must use controlled tenant/topic paths or buckets.
- Kafka events must include tenant and topic context.
- Cache keys must include tenant and topic identifiers where applicable.
- Authorization must occur before document retrieval or RAG generation.
- Logs and metrics must avoid exposing private document content.

Application-supplied tenant identifiers must never be trusted without validating them against the current user's permitted tenant and topics.

## Reliability requirements

The platform should demonstrate:

- Asynchronous ingestion so uploads are not blocked by parsing and embedding.
- Idempotent processing so retrying an event does not create duplicate vectors.
- Retry handling and a failed-event or dead-letter strategy.
- Health, readiness, and liveness checks.
- Persistent storage for stateful services.
- Resource requests and limits.
- Graceful shutdown of API and worker services.
- Horizontal scaling of stateless APIs and Kafka consumers.
- Controlled timeouts between services.
- Backup and restore procedures for PostgreSQL, MinIO, and Qdrant data.

High availability can be demonstrated at the software level, but the local computer remains a single physical failure domain.

## Security requirements

The platform should include:

- Kubernetes Secrets for credentials and API keys.
- No credentials committed to Git.
- Istio mTLS for supported service-to-service communication.
- Authorization checks for every tenant- or topic-scoped operation.
- File type and file size validation.
- Upload quotas and API rate limiting.
- Network policies where practical.
- Non-root containers and restricted security contexts.
- Vulnerability scanning and image scanning in CI.
- Versioned container images rather than the `latest` tag.
- Audit records for important administrative and document actions.

## Observability requirements

Prometheus and Grafana should provide visibility into:

- Request volume, latency, and error rate for each service.
- Istio and Envoy traffic metrics.
- Pod restarts and Kubernetes resource usage.
- Kafka consumer lag and failed events.
- Document ingestion duration and status totals.
- Embedding and model request latency.
- Retrieval latency and number of retrieved chunks.
- Qdrant, PostgreSQL, Redis, and MinIO health.
- Tenant quota and rate-limit activity without exposing document contents.

Distributed tracing and log aggregation can be added after metrics are working.

## GitOps and repository direction

The existing repository contains infrastructure configuration grouped under `charts`, including Istio, Argo CD, and CloudNativePG configuration, along with the kind cluster configuration. Application code and application deployment definitions will be added to the same repository.

A maintainable target structure is:

```text
DOCUMENT-KNOWLEDGE-STORE/
├── apps/                    # Python microservice source code
├── charts/                  # Helm values and platform component configuration
│   ├── argocd/
│   ├── cloudnative-postgres/
│   ├── istio/
│   ├── kafka/
│   ├── minio/
│   ├── qdrant/
│   ├── redis/
│   └── observability/
├── deploy/                  # Application Helm charts or Kubernetes definitions
├── gitops/                  # Argo CD projects and applications
├── kind-cluster/            # Local cluster configuration
├── docs/                    # Architecture decisions and operational guides
└── README.md
```

The structure can evolve gradually. Existing files do not need to be moved until the Argo CD application layout has been selected.

## Delivery phases

### Phase 1: Local platform foundation

- Create the multi-node kind cluster.
- Install Istio base, Istiod, and the Istio ingress gateway.
- Verify local gateway routing.
- Install CloudNativePG and create the PostgreSQL cluster.
- Install Argo CD and verify UI access.
- Register the Git repository with Argo CD.

### Phase 2: Local data platform

- Deploy MinIO through GitOps.
- Deploy Qdrant through GitOps.
- Deploy Redis through GitOps.
- Deploy Kafka through GitOps.
- Confirm persistence and service-to-service connectivity.

### Phase 3: Minimal application path

- Build the Admin/API service.
- Implement topic, user, role, and permission management.
- Implement document upload to MinIO.
- Publish ingestion events to Kafka.
- Build the ingestion worker.
- Create embeddings and store them in Qdrant.
- Build the RAG query service and citation response.

### Phase 4: Operations and security

- Install Prometheus and Grafana.
- Add application and platform dashboards.
- Enable service-mesh security and traffic policies.
- Add quotas, rate limiting, retries, and dead-letter handling.
- Add CI tests, container scans, image builds, and GitOps-based deployments.
- Test failures, recovery, scaling, and backup restoration.

## Current status

The following foundation work has been completed:

- A local multi-node kind cluster has been created.
- Local host-port mapping has been configured for the Istio ingress gateway.
- Istio base has been installed, providing Istio CRDs.
- Istiod has been installed as the service-mesh control plane.
- The Istio ingress gateway has been installed and is running.
- Gateway and VirtualService routing concepts have been tested locally.
- CloudNativePG has been introduced for operator-managed PostgreSQL.
- Argo CD has been installed.
- The Argo CD UI is reachable and administrator login is working.

## Continue from here

### Immediate next milestone: onboard the repository into Argo CD

The next step is **not to deploy another component manually with Helm**. First, connect the existing `DOCUMENT-KNOWLEDGE-STORE` Git repository to Argo CD and define the repository's GitOps entry point.

We should do the following next, one item at a time:

1. Decide the initial `gitops/` directory layout.
2. Create an Argo CD `AppProject` for the platform.
3. Create the first Argo CD `Application` pointing to the repository.
4. Enable automated synchronization, pruning, and self-healing after verifying the target path.
5. Confirm that changing a safe test resource in Git is reflected in the cluster.

Once Git-to-cluster reconciliation is verified, the first new platform component to deploy through Argo CD should be **MinIO**, because it will provide the local object storage required for raw uploaded documents.

**Resume instruction for Codex:** Start by reviewing the current repository tree and designing only the `gitops/` layout and first Argo CD `AppProject`/`Application`. Proceed one step at a time. Do not generate unrelated manifests or move on to MinIO until Argo CD successfully synchronizes the repository.
