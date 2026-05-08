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


### 🧭 5. Trade-offs

wip 

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