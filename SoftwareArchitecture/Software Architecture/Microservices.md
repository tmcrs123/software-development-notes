
## Core principles:

### Decompose by business capabilities

- Each service should own a single business capability (orders service, inventory service, etc...)
- Services should be autonomous and encapsulate both logic and data related to their domain
- Use DDD (domain driven design) principles to define clear service boundaries (Bounded Contexts)

### Independent data stores

Each service should have it's own database as this enforces loose coupling. If one service needs data from another service use event-based communication

## Communication between services

- Prefer async communication using message queues (producer-consumer) or event buses (pub/sub, broadcast).
- If you need to do sync communication use lightweight protocols like HTTP (but this is a very bad idea).
- Do versioning in a backwards compatible way to avoid breaking dependencies

### Service Discovery

Implement service discovery so services can register and discover themselves automatically as manual configuration does not scale

### API Gateway

Use a gateway as a single entry point to route traffic to appropriate services. Gateways should do authentication/authorization, rate limiting, load balancing(meh) and request aggregation (more meh)

###  Use IaC (infrastructure as code)

Treat infrastructure as code to for repeatability, versioning and disaster recovery

### Observability

1. Expose healthchecks
2. Do centralized logging (services log independently but logs end up in central data store)
3. Collect metrics

### Resilience and Fault tolerance

1. Implements retries with back-offs and timeouts,
2. Bulkheads pattern to isolate failures

### CI/CD

- Every service should have independent CI/CD pipelines
- Automated testing from unit, integration and CONTRACT testing

### Security by Design

- Use scoped tokens or service identity systems
- Principle of least privilege
- Secure service-to-service communication (e.g., mTLS).

### Decentralized Governance and Standards

Teams own services end to end but standards (logging, error codes, metrics) need to be defined

### Deployment and Scalability

- Services are independently deployable and scalable
- Use containers and orchestration tools like Kubernetes

## Notes/questions on the above?

What is DDD (Domain-Driven Design)?

**Domain-Driven Design (DDD)** is an approach to software design that focuses on the **core business domain** and its logic.

Key ideas:

- **Model your code around business concepts**, not technical layers.
- Use **"Bounded Contexts"**: separate parts of the system where a specific model applies. Each microservice typically maps to a bounded context.    
- Collaboration between **domain experts** and developers is critical to build accurate models.


In microservices, DDD helps **define clear boundaries** for services so that each one is focused and cohesive.

---

Difference Between Event Buses and Message Queues

Both are used for communication between services, but:

|Feature|**Event Bus**|**Message Queue**|
|---|---|---|
|**Pattern**|Pub/Sub (publish-subscribe)|Point-to-point (producer to consumer)|
|**Receivers**|Multiple subscribers|Typically one consumer per message|
|**Usage**|Broadcasting events (e.g., user signed up)|Task processing (e.g., process order)|
|**Examples**|Kafka, NATS, EventBridge|RabbitMQ, SQS, ActiveMQ|

Use **event buses** for **broadcasting events** (one-to-many), and **queues** for **delegating work** (one-to-one).

---

### 3. **Why Prefer Asynchronous Communication?**

Asynchronous communication is often preferred in microservices because:

- **Decoupling**: Services don’t need to wait for each other.    
- **Resilience**: Failures in one service don’t block others.
- **Scalability**: Messages can be queued and processed at your pace.
- **Performance**: Avoids blocking threads and resources.

Synchronous calls (e.g., HTTP) create **tight coupling** and increase the risk of cascading failures (e.g., one slow service slows down the whole system).

Use synchronous only when immediate responses are required (e.g., login).

---

### 4. **What Are Bulkheads?**

**Bulkheads** are a fault isolation pattern. Think of them like compartments in a ship: if one floods, the others stay afloat.

In microservices:

- You **isolate service resources** (threads, connections) so one misbehaving service can’t exhaust shared resources.
- Helps prevent **cascading failures** across services.

Often used with **thread pools**, **connection pools**, or **rate limiting** per service.

---

### 5. **What Is mTLS (Mutual TLS)?**

**Mutual TLS (mTLS)** is an extension of standard TLS where **both client and server authenticate each other** using certificates.

- **TLS**: Client verifies the server (normal HTTPS).
- **mTLS**: Client and server both present valid certificates.


In microservices:

- It ensures **secure communication** between services.
- Adds **strong authentication**: only trusted services can connect.
    
It’s especially important in **zero-trust architectures** where you can’t assume anything inside the network is safe.

---
### 6. How do you handle stale data?

You don’t **eliminate** stale data in microservices—you **manage and mitigate** it with good patterns.

What does "good patterns" mean? Event driven updates, only one authoritative service (for example if you are doing a checkout process and you need to check if the price of a product hasn't changed then you need to go to the Products service, regardless if you have that data in your microservice database. Only products service holds the truth), cache invalidations, timestamps, etc... 


## 7. How do you handle failing to populate events?

(Assuming you do a true microservices architecture where each service has a database and there's an event bus to which you post events)

The problem where is: what if you do update your database but fail to publish the event? Or what if you publish the event but fail to populate the database? The problem here is that your data is out of sync between services.

The way you fix this is by assuring you always save to the DB and publish the event. How do you do this? With for example a Outbox pattern.

Outbox pattern - the idea is you have a new table called "outbox" that works almost like an email outbox. When you go and change the database you do so in a transaction AND you update the database and the outbox table with the event that just happened.

Then, another process comes and does the work to get the event from the outbox, publish it and delete it. You can now argue "what if this too fails?". Well, in this case there's no way around it. You have to code with the assumption that something wrong (although unlikely will happen). The premise here is that it's better to have you events delivered multiple times rather than not delivered at all (this is called at-least once delivery). Thus, you need to ensure your code works even if the same event is published twice.

## 8. What is the point of doing event sourcing in Microservices

Event sourcing is a pattern that ensures all events are kept in a database. The main idea here is that you have 2 tables, one for events and for state. The state table is derived from the events table - if you replay all events you get the correct value for a piece of state.

If your table grows to a million rows you won't replay all million events. Instead you create an aggregation event which is nothing more than the value of the million events condensed into one and start from there. The advantage is that you save a ton of space, on the flip side you lose history.

History, audit and traceability are the main benefits of event sourcing so condensing events must be a well pondered decision.
