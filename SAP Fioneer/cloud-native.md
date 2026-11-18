# Cloud Native Development – Interview Notes (Java Backend)

> JD line: *"Cloud native development experience on platforms like Azure, AWS, OpenShift (preferable)"*

---

## 1. What is Cloud Native? (30-second answer)

**Cloud native** is an approach to building and running applications that fully exploit cloud computing: they are **containerized, dynamically orchestrated, microservice-oriented, automated and resilient**, so they can scale and change quickly.

> CNCF definition (paraphrased): cloud native technologies let organizations build and run scalable applications in modern dynamic environments (public, private, hybrid) using containers, service meshes, microservices, immutable infrastructure and declarative APIs.

**Say in interview:** *"Cloud native is not just 'running in the cloud'. It's designing apps to be stateless, containerized, horizontally scalable, observable and automatically deployed, so they can survive failures and scale on demand."*

**Cloud-hosted vs cloud-native:** Lifting a monolith onto a VM ("lift and shift") is cloud-hosted. Redesigning it into containerized, independently deployable, auto-scaling services is cloud native.

---

## 2. Core Pillars

| Pillar | Meaning |
|---|---|
| **Microservices** | Small, independently deployable services around business capabilities |
| **Containers** | Package app + dependencies (Docker) for consistent runtime |
| **Orchestration** | Kubernetes/OpenShift handles scheduling, scaling, healing, rollout |
| **DevOps and CI/CD** | Automated build, test, deploy (GitHub Actions, etc.) |
| **Declarative / IaC** | Infrastructure and config as code (Terraform, Helm, YAML) |
| **Observability** | Logs, metrics, traces to understand system behavior |
| **Resilience** | Designed to tolerate failure (retries, circuit breakers, health checks) |
| **Immutable infrastructure** | Replace, don't patch; deploy new image instead of modifying servers |

---

## 3. The 12-Factor App (Very Commonly Asked)

| # | Factor | Practical meaning (Spring Boot) |
|---|---|---|
| 1 | **Codebase** | One repo per service, many deploys |
| 2 | **Dependencies** | Explicit via Maven/Gradle; no reliance on system libs |
| 3 | **Config** | Store in environment, not code (env vars, ConfigMaps, Secrets) |
| 4 | **Backing services** | DB, queue, cache are attached resources via URL/credentials |
| 5 | **Build, release, run** | Strictly separate stages; immutable releases |
| 6 | **Processes** | Stateless; no sticky sessions; state in DB/Redis |
| 7 | **Port binding** | App exposes service via port (embedded Tomcat) |
| 8 | **Concurrency** | Scale out via more instances/processes |
| 9 | **Disposability** | Fast startup, graceful shutdown |
| 10 | **Dev/prod parity** | Keep environments similar (containers, Testcontainers) |
| 11 | **Logs** | Treat as event stream; write to stdout, aggregate externally |
| 12 | **Admin processes** | One-off tasks (migrations) run as separate jobs |

**Memory trick:** *Code, Deps, Config, Backing, Build, Processes, Port, Concurrency, Disposability, Parity, Logs, Admin.*

---

## 4. Containers and Docker (Basics)

**Container vs VM:** Containers share the host OS kernel (lightweight, fast start, MBs); VMs include a full guest OS (heavier, stronger isolation).

**Multi-stage Dockerfile for Spring Boot**
```dockerfile
# Stage 1: build
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B clean package -DskipTests

# Stage 2: run (small image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
USER app                                  # don't run as root
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

**Best practices:** multi-stage builds, slim base images, non-root user, one process per container, `.dockerignore`, pin image versions, scan images (Trivy), use JVM container-aware flags (`MaxRAMPercentage`), layered JARs or Buildpacks (`mvn spring-boot:build-image`).

---

## 5. Kubernetes Essentials

| Object | Purpose |
|---|---|
| **Pod** | Smallest unit; one or more containers sharing network/storage |
| **Deployment** | Manages replicas and rolling updates of Pods |
| **ReplicaSet** | Ensures desired number of Pods (managed by Deployment) |
| **Service** | Stable network endpoint and load balancing for Pods (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP/HTTPS routing from outside to Services |
| **ConfigMap** | Non-sensitive configuration |
| **Secret** | Sensitive data (base64-encoded, enable encryption at rest) |
| **Namespace** | Logical isolation of resources |
| **HPA** | Horizontal Pod Autoscaler: scales replicas on CPU/memory/custom metrics |
| **StatefulSet** | Stable identity/storage for stateful apps (databases) |
| **Job / CronJob** | Batch and scheduled tasks |
| **PersistentVolume / PVC** | Durable storage |

**Minimal Deployment with health probes**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: account-service
spec:
  replicas: 3
  selector:
    matchLabels: { app: account-service }
  template:
    metadata:
      labels: { app: account-service }
    spec:
      containers:
        - name: app
          image: ghcr.io/my-org/account-service:1.0.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef: { name: account-config }
            - secretRef:    { name: account-secret }
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "500m", memory: "1Gi" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
```

**Probes:**
- **Liveness:** is the app alive? If not, restart the container.
- **Readiness:** can it take traffic? If not, remove from Service endpoints.
- **Startup:** for slow-starting apps.

Spring Boot Actuator exposes `/actuator/health/liveness` and `/readiness` (enable `management.endpoint.health.probes.enabled=true`).

---

## 6. Cloud Platforms: Azure, AWS, OpenShift

### Service Mapping (know equivalents)

| Need | AWS | Azure | OpenShift |
|---|---|---|---|
| Managed Kubernetes | **EKS** | **AKS** | OpenShift itself (ROSA on AWS, ARO on Azure) |
| Simple containers | ECS / Fargate | Container Apps / ACI | n/a |
| PaaS for apps | Elastic Beanstalk | App Service | Source-to-Image (S2I) |
| Serverless functions | Lambda | Azure Functions | OpenShift Serverless (Knative) |
| Container registry | ECR | ACR | Internal registry / Quay |
| Managed PostgreSQL | RDS / Aurora | Azure Database for PostgreSQL | Operators / external DB |
| Messaging | SQS / SNS / MSK | Service Bus / Event Hubs | AMQ Streams (Kafka) |
| Secrets | Secrets Manager | Key Vault | Secrets / external vault |
| Object storage | S3 | Blob Storage | via cloud provider |
| Monitoring | CloudWatch | Azure Monitor / App Insights | Built-in Prometheus/Grafana |
| Identity | IAM | Entra ID (Azure AD) | OAuth / RBAC |
| CI/CD | CodePipeline | Azure DevOps | OpenShift Pipelines (Tekton) |
| API gateway | API Gateway | API Management | 3scale |

### What is OpenShift?
Red Hat's **enterprise Kubernetes platform** (Kubernetes + extra features).

| Kubernetes | OpenShift |
|---|---|
| Ingress | **Route** |
| Deployment | Also **DeploymentConfig** (older) |
| Namespace | **Project** |
| Plain pod security | Stricter **Security Context Constraints (SCC)**; containers run as **random non-root UID** by default |
| Build externally | Built-in **S2I** and BuildConfig |
| DIY tooling | Integrated registry, monitoring, CI/CD (Tekton), GitOps (Argo CD), Operators |

**Gotcha to mention:** On OpenShift, images must run as non-root and write only to permitted directories (e.g., `/tmp`), otherwise pods fail due to SCC.

---

## 7. Cloud-Native Patterns for Java Microservices

| Concern | Pattern / Tool |
|---|---|
| **Configuration** | Externalized config: Spring profiles, ConfigMaps/Secrets, Spring Cloud Config, Azure App Configuration, AWS Parameter Store |
| **Service discovery** | Kubernetes DNS/Service (preferred in K8s), Eureka (Spring Cloud) |
| **API gateway** | Spring Cloud Gateway, Kong, cloud API gateways: single entry, auth, rate limiting, routing |
| **Resilience** | **Resilience4j**: circuit breaker, retry, timeout, bulkhead, rate limiter |
| **Messaging / async** | Kafka, RabbitMQ, SQS/Service Bus: decouple services, event-driven |
| **Data consistency** | **Saga** pattern, **Outbox** pattern, eventual consistency (no distributed 2PC) |
| **Database per service** | Each service owns its data |
| **CQRS / Event sourcing** | Separate read/write models; audit-friendly (useful in banking) |
| **Sidecar / Service mesh** | Istio/Linkerd: mTLS, traffic management, retries without code changes |
| **Security** | OAuth2/OIDC + JWT, Spring Security, secrets in vault, mTLS, least privilege |
| **Idempotency** | Idempotency keys so retries are safe (critical for payments) |

### Observability (Three Pillars)

| Pillar | Tools |
|---|---|
| **Logs** | JSON structured logs to stdout, ELK/EFK, Loki, CloudWatch, Azure Monitor |
| **Metrics** | Micrometer, Prometheus, Grafana |
| **Traces** | OpenTelemetry, Zipkin, Jaeger (trace/correlation IDs across services) |

Spring Boot Actuator + Micrometer gives health, metrics and tracing out of the box.

---

## 8. Spring Boot Specifics for Cloud Native

- **Actuator** for health/metrics/info; separate liveness and readiness probes.
- **Graceful shutdown:** `server.shutdown=graceful` + `spring.lifecycle.timeout-per-shutdown-phase=30s`.
- **Externalized config** via env vars (e.g., `SPRING_DATASOURCE_URL`), profiles, ConfigMaps.
- **Stateless** design: no HTTP session in memory; use JWT or Spring Session + Redis.
- **Fast startup / low memory:** Spring Boot 3 supports **GraalVM native images** and **CRaC**; also lazy init, AOT processing.
- **Container images:** Buildpacks (`spring-boot:build-image`) or layered JARs / Jib.
- **Logging:** log to console in JSON.
- **Connection pools** sized according to replicas (replicas × pool size must not exceed DB max connections).

---

## 9. Scalability and Resilience Concepts

- **Horizontal scaling (scale out)** preferred over vertical (scale up) in cloud native.
- **Autoscaling:** HPA (CPU/memory/custom), KEDA (event-driven, e.g., queue length), cluster autoscaler.
- **Fault tolerance:** timeouts everywhere, retries with exponential backoff + jitter (only for idempotent calls), circuit breaker, bulkhead, fallback.
- **High availability:** multiple replicas across availability zones, pod anti-affinity, PodDisruptionBudget.
- **Zero-downtime deploys:** rolling updates with readiness probes, blue-green, canary.
- **Chaos engineering:** deliberately inject failures to validate resilience.

---

## 10. Cloud Service Models

| Model | You manage | Example |
|---|---|---|
| **IaaS** | OS, runtime, app | EC2, Azure VM |
| **PaaS** | App and data | App Service, Elastic Beanstalk, OpenShift |
| **CaaS** | Containers | EKS, AKS |
| **FaaS / Serverless** | Only function code | Lambda, Azure Functions |
| **SaaS** | Nothing | Salesforce, Office 365 |

---

## 11. Infrastructure as Code and GitOps

- **IaC:** Terraform (multi-cloud), CloudFormation (AWS), Bicep/ARM (Azure), Pulumi.
- **Helm:** package manager for Kubernetes (templated charts); **Kustomize** for overlays.
- **GitOps:** Git is the source of truth for desired state; **Argo CD / Flux** sync cluster to repo. Benefits: auditability, easy rollback (git revert), consistency.

---

## 12. Interview Questions and Crisp Answers

**Q1. What does cloud native mean?**
Building apps designed for the cloud: containerized, microservice-based, orchestrated, automated via CI/CD, observable and resilient, scaling horizontally on demand.

**Q2. Cloud native vs lift-and-shift?**
Lift-and-shift moves an existing app unchanged to cloud infrastructure. Cloud native redesigns it to use elasticity, managed services and automation.

**Q3. What are 12-factor apps?**
Methodology for portable, scalable SaaS apps: config in env, stateless processes, logs to stdout, backing services as attached resources, disposability, dev/prod parity, and so on.

**Q4. Docker vs Kubernetes?**
Docker packages and runs containers. Kubernetes orchestrates many containers across nodes: scheduling, scaling, self-healing, service discovery, rolling updates.

**Q5. Liveness vs readiness probe?**
Liveness restarts a stuck container. Readiness controls whether the Pod receives traffic.

**Q6. How do you externalize configuration in Kubernetes?**
ConfigMaps for non-sensitive and Secrets for sensitive values, injected as env vars or mounted files; Spring picks them up through environment properties/profiles. Ideally secrets come from Key Vault/Secrets Manager.

**Q7. How do you make a Spring Boot app cloud ready?**
Stateless, externalized config, Actuator health probes, graceful shutdown, structured stdout logs, container-aware memory settings, metrics and tracing, resilient external calls.

**Q8. How do you handle failures between microservices?**
Timeouts, retries with backoff for idempotent calls, circuit breaker, bulkhead, fallback; async messaging for decoupling; idempotency keys.

**Q9. How do you manage transactions across microservices?**
Avoid distributed transactions; use **Saga** (choreography or orchestration) with compensating actions and **Outbox** pattern for reliable event publishing; accept eventual consistency.

**Q10. How do you scale an application?**
Keep it stateless, set resource requests/limits, use HPA (or KEDA) with replicas across zones, scale the DB layer separately (read replicas, pooling, caching).

**Q11. What is OpenShift and how does it differ from Kubernetes?**
Enterprise Kubernetes distribution from Red Hat adding Routes, S2I builds, stricter security (SCC, non-root), integrated registry, monitoring, Operators, pipelines and a web console.

**Q12. Compare AKS, EKS, OpenShift.**
AKS (Azure) and EKS (AWS) are managed Kubernetes with tight integration into their clouds. OpenShift is an opinionated, batteries-included platform that runs on-prem or on any cloud (also managed as ARO/ROSA), good for hybrid/regulated enterprises.

**Q13. How do you monitor a cloud-native app?**
Logs (structured, centralized), metrics (Micrometer to Prometheus/Grafana or cloud-native monitors), distributed tracing (OpenTelemetry), alerts on SLOs, health endpoints.

**Q14. How do you handle secrets securely?**
Never in code or images; use Key Vault/Secrets Manager/Vault or K8s Secrets with encryption at rest and RBAC; rotate regularly; use workload identity/IAM roles rather than static keys.

**Q15. What is a service mesh and do you always need one?**
A sidecar-based infrastructure layer (Istio/Linkerd) for mTLS, traffic control, retries and telemetry. Not always needed; adds complexity, so adopt when you have many services and need uniform policies.

**Q16. How would you deploy a Spring Boot service to the cloud end to end?**
Build and test with Maven in CI, build a container image, scan it, push to registry (ECR/ACR/GHCR), deploy via Helm/manifests (or GitOps), config from ConfigMaps/Secrets, expose via Ingress/Route, add probes, HPA and monitoring, release with rolling or canary strategy.

**Q17. What is serverless and when would you avoid it?**
Event-driven functions with no server management and pay-per-use. Avoid for long-running tasks, latency-sensitive cold-start scenarios, or heavy JVM startup (unless using native images/SnapStart).

---

## 13. Banking / Financial Context

- Emphasize **security and compliance:** encryption in transit and at rest, audit logs, RBAC, data residency, secrets management.
- **High availability and DR:** multi-AZ/region, backups, RPO/RTO.
- **Idempotency and consistency** for payments; Saga/Outbox, event sourcing for audit trails.
- **Multi-tenancy** and isolation (namespaces, network policies) for SaaS banking platforms.
- **Hybrid cloud** is common, which is one reason OpenShift is popular in financial institutions.

---

## 14. Quick Revision Cheat Sheet

- Cloud native = **containers + microservices + orchestration + CI/CD + observability + resilience**
- **12-factor:** stateless, config in env, logs to stdout, disposable, dev/prod parity
- Docker: **multi-stage, slim, non-root**
- K8s: **Pod → Deployment → Service → Ingress**, ConfigMap/Secret, **HPA**, **liveness/readiness probes**
- Platforms: **EKS / AKS / OpenShift** (Routes, S2I, SCC, non-root)
- Resilience4j: **circuit breaker, retry, timeout, bulkhead**
- Data: **DB per service, Saga, Outbox, idempotency**
- Observability: **logs, metrics (Micrometer/Prometheus), traces (OpenTelemetry)**
- Spring Boot: **Actuator, graceful shutdown, externalized config, native image option**
- IaC/GitOps: **Terraform, Helm, Argo CD**

---

*Next topic suggestions: REST APIs, Spring Boot internals, Hibernate/JPA, OData, Microservices, PostgreSQL, Design Patterns.*