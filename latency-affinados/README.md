# Latency Afficionados

## 🏛️ Structure

### 1. 🎯 Problem Statement and Context

Latency Afficionados is a RETRO video game marketplace where
users can sell RETRO video games and users can also buy such video
games. 

The platform is capable of: 
- posting products
- search products
- view product descriptions
- rating products with review
- comments 
- provide recommendation of products to users based on previews browsing

Our CTO MR Fast want the smallest latencies possible. He cares deeply how things render and how fast they render. 

Right now the whole website is running on React 16. 

You need to find ways to speed up rendering and make sure rendering is fast all times. 

Mr Fast also has a monolith written in Java 1.4 which needs to be migrated to Java 21. 

You need to proposed a decomposition of the monolith.


### 2. 🎯 Goals

1. The application already exists, is needed to plan a migration to a new architecture.
2. Latency should be lower as possible.
3. Cloud-Native approach
4. Application should resilient in case of one component or AZ present failure


### 3. 🎯 Non-Goals


1. one-shoot migration, it usually create many problems, it is hard to rollback, long downtime for customers, and data migration risks/
2. on-premise approach
3. mobile application, it will focus on web application
4. multi-region disaster recovery


### 📐 3. Principles

1. User experience is our main drive. API should respond faster as possible and UI must render with less friction.
2. Low coupling: the components in the architecture should design to dependencies between each other. Design for event-driven architecture.
3. Observability: we need to monitor our APIs specially the response time, also we need to support distributed traces.
4. Scalability: this is a game marketplace so the application should be able to scale fast.


### 🏗️ 4. Overall Diagrams

wip

![](diagrams/overall-architecture.png)


KeyPoints:

- Implementation of the Outbox Pattern to avoid dual-write issues
- Debezium to extract information from the database table
- Product-related events will be published to a Kafka topic
    - The topic will receive multiple product events (product created, product updated, product deleted). This design aims to eliminate any ordering issues
    - Avro for schema definition and versioning
    - Replication Factor = 3
    - Consumers must use manual acknowledgment (manual ack)
- Schema + Avro with multi-event support (https://www.confluent.io/blog/multiple-event-types-in-the-same-kafka-topic/)

Dataloss prevention on Debezium setup

- Debezium uses at-least-once: dont lose data but the publish duplicated
- In Postgres, **replication slots** track the position of consumers from WAL the LSN. Postgres ensure to not discard a segment from WAL until received the acknolodge from all consumers.
- This guaratee have a risk to the WAL grows until reachout out of disk
- Replication Slots should be removed in case of removing some debezium connector
- observability metrics needed:
    - Total Wal Size
    - Debezium LAG
    - Slot Replication Status
    - WAL retained per Replication Slot

links:
- https://www.morling.dev/blog/can-debezium-lose-events/
- https://www.morling.dev/blog/insatiable-postgres-replication-slot/
- https://engineering.zalando.com/posts/2023/11/patching-pgjdbc.html


### 🧭 5. Trade-offs

#### Major Decisions

1. The monolith should be decomposed into domain microservices (products, products-search, rating, comments, recommendation, navigation) 
2. The application should be able to scale fast to attemp spike to usage
3. Communication between services should be designed to avoid dataloss
4. Contract of events should be enforced in the publisher and consumers
5. Product search should fast (100ms) and should not affect or being affected by writes operations
6. The UX should be fluid avoiding/reducing the delay to show the complete page

#### Tradeoffs

**1. Architecture — Microservices vs Modular Monolith**

| Criterion | Microservices | Modular Monolith |
|---|---|---|
| Independent scalability | [good] scale each service to demand, cut cost | [bad] scale the whole app together |
| Fault isolation | [good] failure contained to one service | [careful] fault can affect the entire application |
| Latency (cross-domain flow) | [bad] network hops add latency | [good] local calls (e.g.: method) |
| Operational complexity | [bad] more pipelines, components, monitoring | [good] single deploy unit |
| Deployment time | [good] fast, dont need to deploy the entire application | [bad] very slow, not only the deploy but also the build time |
| Failure modes | [bad] must handle retries, partial failures, eventual consistency | [good] simpler, mostly local failures |

**Decision: Microservices.** Independent scalability and fault isolation (Goals 2 and 4), but it added latency and operational cost. It will be mitigate: cross-service latency with a BFF (trade-off #6) and absorb spikes via K8s autoscaling (trade-off #2).

**2. Compute platform — Kubernetes vs EC2**

| Criterion | Kubernetes (EKS) | EC2 + ASGs |
|---|---|---|
| Autoscaling speed| [good] pods scale in seconds on spikes | [bad] minutes to boot VMs |
| Resilience  | [good] liveness/readiness probes, multi-AZ scheduling | [careful] self-healing must be built ourselves (metrics + ASG) |
| Declarative ops | [good] manifests describe desired state | [careful] more imperative / scripted (shell scripts, ASG templates, etc..) |
| Operational complexity | [bad] needs K8s expertise (many components interacting with each other) | [good] simpler mental model |
| CI Impact | [good] building docker image takes seconds | [bad] building vm takes minutes |

**Decision: Kubernetes (EKS).** It will increase the operational complexity but K8s will enable the platform to fast autoscaling and more resilient due the primitive feature (probes, autoscaling, HPA) to serve spike and AZ-failure tolerance. 

**3. Service communication — Event-Driven vs Request/Response (Sync)**

| Criterion  | Event-Driven (Kafka) | Request/Response (Sync) |
|---|---|---|
| Coupling  | [good] producers/consumers evolve independently | [bad] the client depends from server APIs |
| Resilience | [good] components are not direcly connected; no failure cascade | [bad] downstream outage cascades to caller |
| Scalability  | [good] topic absorbs spikes as back-pressure buffer | [careful] downstream must scale to attend with caller |
| Consistency | [bad] eventual — reads may lag writes | [good] immediate, read-your-writes |
| Debuggability | [careful] async flows are more complex to use distributed tracing | [good] easier to follow a request with distributed tracing|
| Operational cost | [bad] Kafka + schema registry + DLQ to run | [good] no extra infrastructure |

**Decision: Event-Driven (Kafka).** Low coupling and resilience  are core drivers, and the broker act as a back-pressure buffer protecting downstream application when spikes happens. The architecture accept eventual consistency — mitigated by CQRS projection. Observability will be vital to understand this complex architecture and also to be awarn of costs.

**4. Event encoding — Avro Schema vs Plain Text (JSON)**

| Criterion  | Avro + Schema Registry | Plain Text / JSON |
|---|---|---|
| Contract enforcement | [good] validated at publish and consume time | [bad] no enforcement, bad data slips through |
| Schema evolution | [good] backward/forward compat, independent upgrades | [careful] not control, easy to break consumers. It will possible to use shared libraries but it doesnt include a real safety |
| Payload size & latency | [good] compact binary, fast to serialize | [bad] larger, slower to parse |
| Operational cost | [bad] needs schema registry + Avro-aware clients | [good] no extra tooling |
| Bypass risk | [careful] an actor can still publish invalid data directly to the topic | [careful] same risk, plus no schema at all |

**Decision: Avro + Schema Registry.** Enforced contracts and safe schema evolution prevent bad data across independently deployed services, and the compact binary encoding helps latency. The schema-registry tooling dependency is needed, any instability on it will affect the entire application, the resilence of schema-registry is a critical item.

**5. Read/write data model — CQRS vs Single Database**

| Criterion  | CQRS (separate read store) | Single Database |
|---|---|---|
| Search latency  | [good] reads on dedicated store, no write contention | [bad] reads and writes compete |
| Independent scaling | [good] read and write sides scale separately | [bad] one store scales for both |
| Query optimization | [good] denormalized/indexed read model | [careful] schema is a compromise for both | 
| Consistency | [bad] eventual — read store lags until projected (cdc, debezium, ingestion, indexing) | [good] immediate consistency |
| Complexity | [bad] two models to maintain | [good] single model |
| Right Tool for the right problem | [good] we can choose the best database based on write/read requirement | [bad] single choice |

**Decision: CQRS.** Isolating read from writes will allow the system to reduce the latency for the customers. The eventual consistency (Debezium → consumer projection lag) will be the price for lower latency. The cost of maintaining two models is high but we allow to achieve the application goals.

**6. Page composition — BFF vs Frontend calling multiple services**

| Criterion  | BFF (aggregation layer) | Frontend → many services |
|---|---|---|
| Client latency | [good] server-side aggregation, fewer round-trips (round-trips happens but in the private network) | [bad] many client round-trips over the network |
| Payload efficiency | [good] returns exactly what the page needs | [careful] over-fetching, unused fields |
| Decoupling | [good] frontend shielded from downstream changes | [bad] frontend is tied to every service contract |
| Extra component | [bad] another service to build, deploy, monitor | [good] no extra hop |
| Failure surface | [careful] can become a bottleneck / SPOF if not scaled | [good] no shared aggregation point |
| Boundary discipline | [careful] risk of business logic leaking into the BFF | [good] logic stays in domain services |

**Decision: BFF.** Server-side aggregation reduce number of client round-trips and also reduce payload per page. The BFF introduce a high risk since it can be a SPOF/bottleneck, this is a critical service so scaling and monitoring it will be made to ensure the system is working properly. Keep business logic in domain services is critical.



### 🌏 6. For each key major component

wip

### 🖹 7. Migrations

wip

### 🖹 8. Testing strategy

wip 

### 🖹 9. Observability strategy

wip

### 🖹 10. Data Store Designs

wip 

### 🖹 11. Technology Stack

wip

### 🖹 12. References

* wip