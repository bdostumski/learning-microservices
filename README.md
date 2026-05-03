# 🧩 Microservices & Distributed Systems

> A comprehensive, hands-on reference project for **building, containerising, and deploying a cloud-native microservices architecture** from scratch — using Spring Boot, Spring Cloud, Docker, Kubernetes, and RabbitMQ.

![Java](https://img.shields.io/badge/Java-17-blue?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-2.5.7-brightgreen?logo=springboot)
![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2020.0.3-brightgreen?logo=spring)
![Docker](https://img.shields.io/badge/Docker-Compose-blue?logo=docker)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Minikube-326CE5?logo=kubernetes)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.9-orange?logo=rabbitmq)
![Zipkin](https://img.shields.io/badge/Zipkin-Distributed%20Tracing-yellow)

---

## 📖 Overview

This project walks through every stage of building a production-style microservices system:

1. **Maven multi-module project** structure with a parent POM
2. **Synchronous communication** via REST Template → Eureka Service Discovery → OpenFeign
3. **Asynchronous messaging** with RabbitMQ (AMQP)
4. **Distributed tracing** with Spring Cloud Sleuth + Zipkin
5. **API Gateway** routing with Spring Cloud Gateway
6. **Containerisation** using Google Jib (no Dockerfile needed)
7. **Orchestration** with Kubernetes (Minikube locally, Linode in cloud)
8. **Security** with API Key authentication at the gateway level

---

## 🏗️ Architecture

!["Software Architecture"](./resources/architecture_main.jpg)

```
                        ┌─────────────────┐
           HTTP         │   API Gateway   │  :8083
Client ───────────────▶ │     (apigw)     │
                        └────────┬────────┘
                                 │ routes via Eureka / K8s DNS
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐
       │  Customer   │   │    Fraud    │   │  Notification    │
       │  Service    │   │   Service   │   │    Service       │
       │   :8080     │   │   :8081     │   │    :8082         │
       └──────┬──────┘   └──────┬──────┘   └────────┬─────────┘
              │  Feign (sync)   │                    ▲
              └─────────────────┘           RabbitMQ │ (async)
                                           ┌─────────┴────────┐
                                           │     RabbitMQ     │
                                           │  :5672 / :15672  │
                                           └──────────────────┘

       ┌──────────────────┐     ┌──────────────┐     ┌────────────────┐
       │  Eureka Server   │     │   Zipkin UI  │     │   PostgreSQL   │
       │    :8761         │     │    :9411     │     │    :5432       │
       └──────────────────┘     └──────────────┘     └────────────────┘
```

---

## 📦 Modules

| Module | Description | Port |
|---|---|---|
| `customer` | Customer registration service. Validates registrations, calls Fraud service synchronously, and publishes async events to RabbitMQ | `8080` |
| `fraud` | Fraud detection service. Called synchronously by Customer via OpenFeign; checks if a customer registration is fraudulent | `8081` |
| `notification` | Notification service. Listens on a RabbitMQ queue and persists notification records to the database asynchronously | `8082` |
| `apigw` | Spring Cloud API Gateway — single external entry point. Routes requests to downstream services and integrates with Eureka for load balancing | `8083` |
| `eureka-server` | Netflix Eureka service registry. All microservices register here so they can discover and call each other without hardcoded hostnames/ports | `8761` |
| `clients` | Shared library of OpenFeign client interfaces used by other services for inter-service HTTP calls | — |
| `amqp` | Shared RabbitMQ/AMQP configuration library. Provides `RabbitMQMessageProducer` and Jackson message converter setup | — |

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Language | Java 17 | Core language (Eclipse Temurin JRE in containers) |
| Framework | Spring Boot 2.5.7 | Application framework |
| Service Discovery | Spring Cloud Netflix Eureka | Dynamic service registration & lookup |
| API Gateway | Spring Cloud Gateway | Single entry point, path-based routing, load balancing |
| Inter-service calls | Spring Cloud OpenFeign | Declarative REST client with Eureka integration |
| Async messaging | RabbitMQ 3.9 (AMQP 0-9-1) | Decoupled async communication between services |
| Distributed tracing | Spring Cloud Sleuth + Zipkin | End-to-end request tracing across services |
| Persistence | PostgreSQL | Relational database for all services |
| DB Admin UI | pgAdmin 4 | Web-based PostgreSQL management |
| Caching | Caffeine | In-memory cache (shared via parent POM) |
| Build | Maven (multi-module) | Multi-module project management & build lifecycle |
| Containerisation | Google Jib | Builds optimised Docker images without a Dockerfile or Docker daemon |
| Orchestration | Kubernetes (Minikube / Linode) | Container orchestration, scaling, zero-downtime deploys |
| Boilerplate | Lombok | Reduces boilerplate (getters, builders, logging) |

---

## 🚀 Getting Started

### Prerequisites

- **Java 17** (Eclipse Temurin recommended)
- **Maven 3.8+**
- **Docker & Docker Compose**
- _(Optional)_ **Minikube + kubectl** for Kubernetes deployment
- _(Optional)_ **Docker Hub account** for pushing images with Jib

---

### 1. Start Infrastructure Services

Start PostgreSQL, pgAdmin, RabbitMQ and Zipkin locally:

```bash
docker compose up -d
```

| Service | URL | Credentials |
|---|---|---|
| pgAdmin | http://localhost:5050 | `b.dostumski@syscomz.com` / `password` |
| RabbitMQ Management | http://localhost:15672 | `guest` / `guest` |
| Zipkin UI | http://localhost:9411 | — |
| PostgreSQL | `localhost:5432` | `syscomz` / `password` |

---

### 2. Run Services Locally (Development)

```bash
# Build all modules from the root
mvn clean install -DskipTests

# Start each service (in order)
cd eureka-server && mvn spring-boot:run
cd apigw        && mvn spring-boot:run
cd customer     && mvn spring-boot:run
cd fraud        && mvn spring-boot:run
cd notification && mvn spring-boot:run
```

Or use `java -jar`:
```bash
java -jar customer/target/customer-1.0-SNAPSHOT.jar
```

> 💡 **Tip:** To run multiple instances of a service (for load balancing testing), duplicate the run configuration in your IDE and add `--server.port=8085` to Program Arguments.

---

### 3. Run Full Stack with Docker Compose

Uses pre-built images from Docker Hub (includes all microservices):

```bash
docker-compose -f docker-compose-v1.yml up -d
```

| Service | URL |
|---|---|
| Customer API | http://localhost:8080 |
| Fraud API | http://localhost:8081 |
| Notification API | http://localhost:8082 |
| API Gateway | http://localhost:8083 |
| Eureka Dashboard | http://localhost:8761 |

> 💡 Use `docker-compose pull` first to get the latest images from Docker Hub.

---

### 4. Build & Push Docker Images (Jib)

Images are built and pushed to Docker Hub automatically during `mvn package` via Google Jib. No Dockerfile required.

```bash
# Build & push all microservices
mvn clean package

# Build & push only the API Gateway (using a Maven profile)
cd apigw
mvn clean package -P build-docker-image
```

Make sure you're logged in before pushing:
```bash
docker login
```

Images are published as `bdostumski/<service-name>:latest`, e.g. `bdostumski/customer:latest`.

> ⚠️ If `mvn -version` shows the wrong Java version, fix it with:
> ```bash
> export JAVA_HOME=$(/usr/libexec/java_home -v 17)
> ```

---

### 5. Deploy to Kubernetes (Minikube)

```bash
# Start Minikube with enough memory
minikube start --memory=14g

# Apply bootstrap resources (namespaces, PVCs, etc.)
kubectl apply -f k8s/minikube/bootstrap/

# Deploy all services
kubectl apply -f k8s/minikube/services/

# Expose LoadBalancer services (required for Minikube)
minikube tunnel
```

> ℹ️ When running on Kubernetes, **Eureka is disabled** — K8s DNS handles service discovery natively.

---

### 6. Cloud Deployment (Linode Kubernetes)

```bash
# Set kubeconfig to your Linode cluster
export KUBECONFIG=~/syscomz-kubeconfig.yml

# Apply all resources
kubectl apply -f k8s/minikube/bootstrap/
kubectl apply -f k8s/minikube/services/

# Verify
kubectl get po
kubectl get svc
```

---

## 🔍 Distributed Tracing with Sleuth + Zipkin

Every service is instrumented with **Spring Cloud Sleuth**, which automatically injects and propagates a `TraceId` and `SpanId` into every log line and across HTTP/AMQP boundaries.

**Example log output:**
```
[customer, a86f2368573ce95f, a86f2368573ce95f]  ← same TraceId, root Span
[fraud,    a86f2368573ce95f, 4bdf86c263154fe5]  ← same TraceId, new Span
[notification, a86f2368573ce95f, c64809a52a015a37] ← same TraceId, new Span
```

All trace data is forwarded to **Zipkin** for visualisation:

👉 Open [http://localhost:9411](http://localhost:9411) and click **Run Query** to view traces.

!["Zipkin UI"](./resources/zipkin_ui.png)
!["Zipkin UI Dependencies"](./resources/zipkin_ui_dependencies.png)

---

## 🐇 Asynchronous Messaging with RabbitMQ

The **Customer** service publishes notification events to a RabbitMQ exchange. The **Notification** service listens on a dedicated queue and persists the message to the database — completely decoupled.

**AMQP Exchange types used:**
| Type | Behaviour |
|---|---|
| **Direct** | Routes to queues where `routing-key == binding-key` |
| **Fanout** | Broadcasts to all bound queues |
| **Topic** | Partial match routing (e.g. `foo.*` matches `foo.bar`) |
| **Headers (Default/Nameless)** | Routes where `routing-key == queue name` |

!["RabbitMQ Architecture"](./resources/rabbitmq_architecture.png)
!["RabbitMQ Messages Exchange"](./resources/rabbitmq_messages_exchange.png)

> 💡 If the Notification service goes down, RabbitMQ **holds the messages** in the queue until the service recovers. Messages are only removed after the consumer sends an ACK.

**When to use RabbitMQ vs Kafka:**
- **RabbitMQ** — best for task queues, complex routing, inter-service messaging, and scenarios where messages should be removed once consumed.
- **Kafka** — best for high-throughput event streaming, data pipelines, replay scenarios, and audit logs.

---

## 🌐 API Gateway & Load Balancing

The **apigw** module uses Spring Cloud Gateway to route all external traffic. It integrates with Eureka for client-side load balancing using the **Round Robin** algorithm (requests distributed sequentially across service instances).

**Key gateway responsibilities:**
- **Path-based routing** — e.g. `/api/v1/customers/**` → `customer` service
- **Load balancing** across multiple service instances
- **TLS termination** (delegated to cloud provider in production)
- **Authentication** (API Key filter — see Security section)

> 💡 In production, delegate the external load balancer to your cloud provider (AWS ALB, GCP Cloud Load Balancing, Azure Application Gateway). Use Spring Cloud Gateway only as an internal ingress.

!["Internal and External Load Balancers"](./resources/load_balancer.png)
!["Load Balancer Algorithms"](./resources/load_balancer_algorithms.png)
!["Load Balancer Health Checks"](./resources/load_balancer_health_checks.png)

---

## 🔒 Security — API Key Authentication

The gateway enforces API Key authentication. Each client presents a key which is validated at the `apigw` filter before the request is forwarded to the private network.

!["Security Architecture"](./resources/spring-security.png)
!["API Key DB Schema"](./resources/spring-security-api-key-db-schema.png)
!["Security Filter Check"](./resources/spring-security-filter-check.png)

---

## ☸️ Kubernetes Concepts

### Cluster Architecture

```
Master Node (Control Plane)            Worker Nodes
┌──────────────────────────────┐       ┌─────────────────────┐
│  API Server  ← kubectl       │       │  Kubelet            │
│  Scheduler                   │◀─────▶│  Container Runtime  │
│  Controller Manager          │       │  Kube Proxy         │
│  etcd (cluster state)        │       │  Pods (our apps)    │
│  Cloud Controller Manager    │       └─────────────────────┘
└──────────────────────────────┘
```

| Component | Role |
|---|---|
| **API Server** | Frontend for the control plane; all kubectl commands go here |
| **Scheduler** | Assigns new pods to nodes based on resource availability |
| **etcd** | Distributed key-value store; single source of truth for cluster state |
| **Controller Manager** | Runs Node, ReplicaSet, Endpoint, and other controllers to reconcile desired vs actual state |
| **Cloud Controller** | Integrates with cloud provider for load balancers, storage, and VMs |
| **Kubelet** | Agent on each node; manages pod lifecycle |
| **Kube Proxy** | Handles network routing rules on each node |

!["K8s Control Plane"](./resources/k8s-control-plane.png)
!["K8s Pods"](./resources/k8s-pods.png)
!["K8s Controller Manager"](./resources/k8s-controller-manager.png)

### Key K8s Resources Used

| Resource | Purpose |
|---|---|
| **Deployment** | Manages rolling updates and ReplicaSets for stateless services |
| **StatefulSet** | Used for PostgreSQL (stable network identity + persistent storage) |
| **Service (ClusterIP)** | Internal service discovery between pods |
| **Service (LoadBalancer)** | Exposes services externally (provisioned by cloud or `minikube tunnel`) |
| **ConfigMap** | Stores non-sensitive environment configuration |

> ⚠️ **Never deploy databases inside your K8s cluster in production.** Use managed services like Amazon RDS or Linode Managed Databases instead.

!["K8s Deployment Architecture"](./resources/k8s-deployment-architecture.png)
!["K8s kubectl get all"](./resources/k8s-kubectl-get-all.png)

---

## 📁 Project Structure

```
learning-microservices/
├── amqp/                       # Shared RabbitMQ/AMQP configuration module
├── apigw/                      # Spring Cloud API Gateway
├── clients/                    # Shared OpenFeign client interfaces
├── customer/                   # Customer microservice
├── eureka-server/              # Netflix Eureka service registry
├── fraud/                      # Fraud detection microservice
├── notification/               # Notification microservice (async, via RabbitMQ)
├── resources/                  # Architecture diagrams and screenshots
├── k8s/
│   └── minikube/
│       ├── bootstrap/          # Namespace, PVC, ConfigMap manifests
│       └── services/           # Deployment & Service manifests per microservice
├── docker-compose.yml          # Infrastructure only (PostgreSQL, pgAdmin, RabbitMQ, Zipkin)
├── docker-compose-v1.yml       # Full stack (all microservices + infrastructure)
├── docker-compose-v3.yml       # Alternative compose variant
└── pom.xml                     # Parent Maven POM (dependency & plugin management)
```

---

## 📋 Cheat Sheet

### Application Commands

```bash
mvn spring-boot:run                              # Run a service with Spring Boot
java -jar target/service-1.0-SNAPSHOT.jar        # Run a packaged JAR directly
mvn clean package -P build-docker-image          # Build & push a Docker image via Jib
mvn clean package                                # Build & push all images (Jib, from root)
```

### Maven Lifecycle

| Phase | Description |
|---|---|
| `validate` | Validate project configuration |
| `compile` | Compile source code |
| `test` | Run unit tests |
| `package` | Package into a JAR |
| `verify` | Run integration test checks |
| `install` | Install JAR to local `.m2` repository |
| `deploy` | Deploy JAR to remote repository |

### Docker Commands

```bash
docker compose up -d                             # Start all infrastructure containers
docker compose -f docker-compose-v1.yml up -d   # Start full stack
docker-compose pull                              # Pull latest images from Docker Hub
docker logs <container_name>                     # View container logs
docker network ls                                # List Docker networks
docker volume ls                                 # List Docker volumes
docker network prune                             # Remove unused networks
docker volume prune                              # Remove unused volumes
docker info                                      # Show Docker host info (CPU, RAM)
docker network create postgres                   # Create the postgres bridge network
docker run -d -p 9411:9411 openzipkin/zipkin     # Run Zipkin as a standalone container
```

### kubectl / Minikube Commands

```bash
minikube start --memory=14g                      # Start Minikube with 14 GB RAM
minikube status                                  # Show cluster status
minikube ip                                      # Get master node IP
minikube tunnel                                  # Expose LoadBalancer services locally
minikube ssh                                     # SSH into the Minikube node
minikube service --url <service_name>            # Get the URL for a service

kubectl apply -f k8s/minikube/bootstrap/         # Apply all bootstrap manifests
kubectl apply -f k8s/minikube/services/          # Deploy all microservice manifests
kubectl delete -f <path>                         # Delete resources from a manifest
kubectl get all                                  # List all pods, services, deployments
kubectl get pods                                 # List running pods
kubectl get svc                                  # List services
kubectl describe pod <pod_name>                  # Show detailed pod information
kubectl logs <pod_name>                          # Print pod logs
kubectl logs -f <pod_name>                       # Follow pod logs (tail -f)
kubectl port-forward pod/<pod_name> 8080:80      # Forward pod port to localhost
kubectl delete pod <pod_name>                    # Delete a pod (will be recreated)
kubectl scale --replicas=0 deployment <name>     # Scale a deployment to 0 instances
kubectl exec -it postgres-0 -- psql -U syscomz  # Open psql CLI in the postgres pod
```

### Maven Multi-Module Archetypes

```bash
mvn archetype:generate \
  -DgroupId=com.syscomz \
  -DartifactId=syscomzservices \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false
```

---

## 📚 Reference Links

### Core Technologies
- [Spring Cloud](https://spring.io/projects/spring-cloud) — Distributed systems patterns for Spring Boot
- [Spring Cloud Netflix (Eureka)](https://spring.io/projects/spring-cloud-netflix) — Service Discovery
- [Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign) — Declarative REST client
- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway) — API Gateway & internal load balancer
- [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth) — Distributed tracing auto-configuration
- [Spring Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.profiles) — Environment-specific configuration

### Messaging
- [RabbitMQ AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts.html)
- [Apache Kafka](https://kafka.apache.org/) — Distributed event streaming platform
- [Amazon SQS](https://aws.amazon.com/sqs/) — Fully managed cloud message queuing
- [When to use RabbitMQ over Kafka?](https://stackoverflow.com/questions/42151544/when-to-use-rabbitmq-over-kafka)

### Tracing & Observability
- [Zipkin](https://zipkin.io/) — Distributed tracing system and UI
- [OpenTracing](https://opentracing.io/) — Distributed tracing standards

### Build & Containerisation
- [Apache Maven](https://maven.apache.org/index.html) — Project management & build tool
- [Maven Compiler Plugin](https://maven.apache.org/plugins/maven-compiler-plugin/)
- [Spring Boot Maven Plugin](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/htmlsingle/)
- [Maven Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Jib — Containerise Java Apps](https://github.com/GoogleContainerTools/jib) — Build Docker images without a Dockerfile
- [Eclipse Temurin (OpenJDK images)](https://hub.docker.com/_/eclipse-temurin/)
- [Docker Resource Constraints](https://docs.docker.com/config/containers/resource_constraints/)
- [Remove Docker Images, Containers and Volumes](https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes)

### Kubernetes
- [Kubernetes Official Docs](https://kubernetes.io/)
- [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [Install Kubernetes Tools](https://kubernetes.io/docs/tasks/tools/)
- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Spring Cloud Kubernetes PropertySource](https://docs.spring.io/spring-cloud-kubernetes/docs/current/reference/html/#kubernetes-propertysource-implementations)

### Cloud Deployment
- [Amazon EKS](https://aws.amazon.com/eks/) — Managed Kubernetes on AWS
- [Amazon ECR](https://aws.amazon.com/ecr/) — Container image registry
- [Amazon RDS](https://aws.amazon.com/rds/) — Managed relational databases
- [Amazon MQ](https://aws.amazon.com/amazon-mq/) — Managed RabbitMQ / ActiveMQ
- [Amazon MSK](https://aws.amazon.com/msk/) — Managed Apache Kafka
- [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- [GCP Cloud Load Balancing](https://cloud.google.com/load-balancing)
- [NGINX Load Balancer](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)

### Security & Secrets
- [HashiCorp Vault](https://www.vaultproject.io/) — Secrets management
- [Spring Vault](https://spring.io/projects/spring-vault)

### Tooling
- [diagrams.net](https://www.diagrams.net/) — Architecture diagram tool
- [Spring Boot Banner Generator](https://devops.datenkollektiv.de/banner.txt/index.html)
- [Maven Archetype Guide](https://maven.apache.org/guides/mini/guide-creating-archetypes.html)

---

## 📝 Learning Notes

### Maven Multi-Module Project
> A multi-module project is built from an aggregator POM that manages a group of submodules. The aggregator lives in the root directory with `<packaging>pom</packaging>`. Submodules can be built independently or together through the root POM — reducing duplication and centralising dependency management.

**Setup steps:**
1. Generate parent module with `mvn archetype:generate`
2. Delete the `src/` folder from the parent (it's an aggregator only)
3. Configure `<modules>` and `<dependencyManagement>` in the parent `pom.xml`
4. Add `<packaging>jar</packaging>` to each child module

### Service Communication Evolution in This Project

| Step | Method | Notes |
|---|---|---|
| 1 | `RestTemplate` | Hardcoded `http://localhost:8081/...` — brittle, not scalable |
| 2 | `RestTemplate` + `@LoadBalanced` + Eureka | Uses service name `http://FRAUD/...` — dynamic, load balanced |
| 3 | `OpenFeign` + Eureka | Declarative interface, cleanest approach |
| 4 | RabbitMQ (async) | For fire-and-forget events (e.g. notifications) |
| 5 | K8s DNS | Replaces Eureka entirely in Kubernetes environments |

### Eureka Service Discovery Flow

!["Eureka Server Communication"](./resources/eureka_server_communication.png)
!["Eureka Server Communication Example"](./resources/eureka_server_communication_example.png)

1. **Register** — each service registers its host/port with Eureka on startup
2. **Lookup** — when Service A needs to call Service B, it queries Eureka
3. **Connect** — Service A connects directly to Service B using the discovered address

### Docker Networking & Spring Profiles

When services run inside Docker containers, `localhost` no longer refers to another container. Fix:
1. Define a shared Docker network (e.g. `spring`) in `docker-compose.yml`
2. Use the **container name** as the hostname in config (e.g. `rabbitmq`, `postgres`)
3. Create an `application-docker.yml` profile in each service with container hostnames
4. Set `SPRING_PROFILES_ACTIVE=docker` in the Docker Compose environment

### Jib Image Build Flow

```
Source Code → Maven Package → Jib Plugin → Docker Image → Docker Hub Registry
```

No Docker daemon needed. Jib builds layered, reproducible images directly from the classpath.

### Kubernetes vs Eureka

| Feature | Eureka (Spring Cloud) | Kubernetes |
|---|---|---|
| Service registry | Eureka Server pod | Built-in etcd + kube-dns |
| Load balancing | Ribbon (client-side) | kube-proxy + Service (server-side) |
| Health checks | Eureka heartbeat | Liveness & Readiness probes |
| Usage | Local / Docker Compose | K8s environments |

> In this project, Eureka is **disabled** (`eureka.client.enabled: false`) when deploying to Kubernetes.

### Kubernetes Deployment Checklist

```bash
# 1. Build and push Docker images
mvn clean package

# 2. Start Minikube
minikube start --memory=14g

# 3. Apply bootstrap (databases, config)
kubectl apply -f k8s/minikube/bootstrap/

# 4. Log into the postgres pod and create databases
kubectl exec -it postgres-0 -- psql -U syscomz

# 5. Deploy microservices
kubectl apply -f k8s/minikube/services/

# 6. Expose LoadBalancers
minikube tunnel

# 7. Test via API Gateway external IP
kubectl get svc   # grab EXTERNAL-IP for apigw
```

---

## 🔮 Further Reading & Next Steps

| Topic | Resource |
|---|---|
| Config management | [Spring Cloud Config](https://spring.io/projects/spring-cloud-config) or [Kubernetes ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Secrets management | [HashiCorp Vault](https://www.vaultproject.io/) + [Spring Vault](https://spring.io/projects/spring-vault) |
| Reporting / Analytics | Dedicated read-model microservice with optimised queries (avoid long JPA queries across services) |
| Production DB | [Amazon RDS](https://aws.amazon.com/rds/) or Linode Managed Databases — never run DBs inside K8s in production |
| Production MQ | [Amazon MQ](https://aws.amazon.com/amazon-mq/) or [Amazon MSK](https://aws.amazon.com/msk/) |

---

## 👤 Author

**Borislav Dostumski**  
🐙 [@bdostumski](https://github.com/bdostumski)

---

*Built for learning purposes — contributions and feedback are welcome!*
