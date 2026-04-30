# Software Architecture Patterns — Complete Deep Dive
> Created by @kriti.sde | All content original.

---

## What is Software Architecture?

Software Architecture is the **highest level decision** about how a system is structured.

It answers three fundamental questions:
- **What** components exist in the system?
- **How** do they communicate with each other?
- **Why** were these decisions made over alternatives?

Architecture is decided **before** you draw a single HLD diagram. It is the blueprint of the blueprint.

### Architecture vs HLD vs LLD

| Level | What it answers | Example |
|---|---|---|
| **Architecture** | The WHAT and WHY | "We will build this as microservices" |
| **HLD** | The HOW at a high level | "Here are the components and data flow" |
| **LLD** | The HOW at a low level | "Here is the class, schema, algorithm" |

### Real World Architecture Decisions

- **Netflix** said → "We will be distributed and microservices based." That is an architectural decision.
- **WhatsApp** said → "We will be event driven for messaging." That is an architectural decision.
- **Bitcoin** said → "We will be peer to peer with no central server." That is an architectural decision.

---

## The Complete Hierarchy

Before listing patterns — understand that not all patterns are the same type of thing.

```
ARCHITECTURAL STYLES
│
├── STRUCTURAL — how your system is organized
│   ├── Monolithic
│   ├── Client-Server
│   ├── Layered / N-Tier
│   │     Note: MVC is layered thinking applied to app code
│   ├── Microkernel
│   └── Hexagonal
│
├── DISTRIBUTED — how your system is spread
│   ├── SOA (Service Oriented Architecture)
│   │     Note: Microservices evolved from SOA
│   ├── Microservices
│   ├── Space-Based
│   └── Broker
│
├── COMMUNICATION — how your parts talk
│   ├── Event-Driven
│   │     Note: CQRS lives inside Event-Driven
│   ├── Peer-to-Peer
│   └── Pipe-Filter
│
└── INFRASTRUCTURE — how your system runs
    ├── Serverless
    └── Master-Slave
```

### What Does NOT Belong Here

| Pattern | Why it doesn't belong | Where it actually belongs |
|---|---|---|
| MVC | Application design pattern, not system architecture | Frontend / Backend design |
| Circuit Breaker | Failure resilience pattern | Failure patterns |
| Blackboard | Niche AI / expert systems pattern | Advanced AI systems |

---

## Category 1 — Structural Patterns

> **Core question: How should I organize my system?**

These patterns define where each piece of code lives and how the system is physically and logically structured.

---

### 1.1 Monolithic Architecture

**Tag: Beginner**

#### Definition
A monolithic architecture is a single unified unit where all components — UI, business logic, and database access — are packaged and deployed together as one application.

#### How it looks
```
[ Single Application ]
        │
        ├── UI Layer
        ├── Business Logic Layer
        ├── Data Access Layer
        │
        └── [ Single Database ]
```

#### Characteristics
- One codebase
- One deployment unit
- All modules share the same process and memory
- Changes to any part require redeploying the whole application

#### Real World Examples
- Early Instagram (before 2012)
- Early Uber (before splitting into microservices)
- Early Amazon (Jeff Bezos famously started with a monolith)
- Stack Overflow (still largely monolithic — and handles massive scale)

#### When to use
- You are starting a new product
- Small team (less than 10 engineers)
- Business requirements are still changing fast
- Speed of development matters more than scale

#### When NOT to use
- Multiple large teams need to work independently
- Different parts of the system need to scale differently
- You need to deploy parts of the system independently

#### Strengths
- Simple to develop, test and debug
- Easy to deploy — one thing to ship
- No network latency between components
- Shared memory makes data access fast
- Lower operational complexity

#### Weaknesses
- Scales as one unit — you cannot scale just the checkout service
- One bug in any module can crash the entire application
- Codebase becomes harder to understand as it grows
- Technology lock-in — hard to switch one part to a different language
- Deployment risk increases — every release touches everything

#### Evolution Path
```
Monolith → gets too big → Layered Monolith → gets too complex → Microservices
```

---

### 1.2 Client-Server Architecture

**Tag: Beginner**

#### Definition
A distributed architecture where responsibilities are split between service providers (servers) and service requesters (clients). The client initiates requests and the server processes and responds.

#### How it looks
```
[ Client ]  ──── Request ────▶  [ Server ]
[ Browser ]                     [ Backend ]
[ Mobile App ]  ◀──── Response ───  [ API ]
                                    │
                                [ Database ]
```

#### Characteristics
- Clear separation of concerns between client and server
- Client handles presentation and user interaction
- Server handles business logic and data
- Communication happens over a network (HTTP, WebSocket, etc.)

#### Real World Examples
- Every website you have ever visited
- Gmail — browser (client) talks to Google servers
- Spotify — app (client) streams from Spotify servers
- Any mobile app with a backend

#### Variants
**Two-tier:** Client talks directly to database server. Simple but insecure.

**Three-tier:** Client talks to application server which talks to database. Most common pattern today.

**N-tier:** Multiple layers between client and data. Used in enterprise systems.

#### When to use
- Almost always — this is the foundation pattern
- Any system where users need to interact with centralized data
- When you need to control data on the server side

#### Strengths
- Clear responsibility separation
- Server can be updated without changing clients
- Centralized data management and security
- Multiple clients (web, mobile, desktop) can share one server

#### Weaknesses
- Server becomes a bottleneck under high load
- Single point of failure if server goes down
- Network latency between client and server
- Server must handle all concurrent client connections

---

### 1.3 Layered Architecture (N-Tier)

**Tag: Beginner**

#### Definition
Organizes the system into horizontal layers where each layer has a specific responsibility and only communicates with the layer directly below it. Requests flow down through layers, responses flow back up.

#### How it looks
```
┌─────────────────────────────┐
│     Presentation Layer      │  ← UI, API endpoints
├─────────────────────────────┤
│    Business Logic Layer     │  ← Rules, workflows, calculations
├─────────────────────────────┤
│     Data Access Layer       │  ← Queries, ORM, repositories
├─────────────────────────────┤
│       Database Layer        │  ← Actual data storage
└─────────────────────────────┘
```

#### The Golden Rule
**Each layer only talks to the layer directly below it.**

Presentation never talks directly to Database. Business logic never skips to Database. Every request travels through every layer in order.

#### MVC — Layered Thinking Applied to Application Code

MVC (Model-View-Controller) is NOT a separate architectural pattern. It is layered architecture applied at the application code level:

- **Model** = Data layer (what layered calls "data access layer")
- **View** = Presentation layer (what layered calls "presentation layer")
- **Controller** = Business logic layer (what layered calls "business logic layer")

Same concept. Smaller scope. MVC organizes your code. Layered organizes your system.

#### Real World Examples
- Banking and financial systems
- Enterprise resource planning (ERP) software
- Government and healthcare systems
- Most traditional corporate web applications

#### When to use
- When different teams own different layers
- When you need clear separation of concerns
- When maintainability and organization matter
- Enterprise systems with strict compliance requirements

#### Strengths
- Highly organized and maintainable
- Each layer is independently testable
- Easy to replace one layer without touching others
- Clear ownership — frontend team owns presentation, backend team owns logic

#### Weaknesses
- Every request passes through all layers — adds latency
- Can become overly rigid and slow to change
- "Sinkhole anti-pattern" — requests pass through layers that add no value
- Not suitable for high performance real-time systems

---

### 1.4 Microkernel Architecture

**Tag: Intermediate**

#### Definition
A core system (the microkernel) provides minimal base functionality. All additional features are implemented as plug-in components that attach to the core. The core knows about plugins. Plugins do not know about each other.

#### How it looks
```
        ┌─────────────────┐
        │   Microkernel   │  ← Core system, minimal functionality
        │   (Core System) │
        └────────┬────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│Plugin │   │Plugin │   │Plugin │  ← Independent feature modules
│  A    │   │  B    │   │  C    │
└───────┘   └───────┘   └───────┘
```

#### Characteristics
- Core system is lean and stable
- Features are added as plugins without touching the core
- Plugins can be added, removed, or updated independently
- Core defines the plugin contract (interface)

#### Real World Examples
- **VS Code** — core editor + thousands of extensions
- **Eclipse IDE** — everything beyond basics is a plugin
- **Chrome / Firefox** — browser extensions
- **WordPress** — core CMS + plugins for every feature
- **Jenkins** — core CI server + build plugins

#### When to use
- Systems where features need to be added or removed at runtime
- Products with a marketplace of third-party extensions
- When different customers need different feature sets
- Tools and development environments

#### Strengths
- Extremely extensible without modifying core
- Plugins can be developed by third parties
- Core remains stable and reliable
- Features can be enabled/disabled per customer

#### Weaknesses
- Plugin interfaces are difficult to change once established
- Plugins can conflict with each other
- Performance overhead of plugin communication
- Core can become a bottleneck if not designed carefully

---

### 1.5 Hexagonal Architecture (Ports and Adapters)

**Tag: Advanced**

#### Definition
The business logic (domain) sits at the center of the system and knows absolutely nothing about the outside world. It communicates through Ports (interfaces it defines). Adapters implement those ports to connect to databases, UIs, APIs, queues, or any external system.

#### How it looks
```
                    [ UI Adapter ]
                          │
[ Queue Adapter ] ──── [ PORT ] ──── [ REST API Adapter ]
                          │
                  [ Business Logic ]   ← knows nothing outside
                          │
[ Test Adapter ] ──── [ PORT ] ──── [ Database Adapter ]
                          │
                    [ Cache Adapter ]
```

#### Core Concept: Dependency Inversion
Normal architecture: Business logic depends on database.
Hexagonal architecture: Database adapter depends on business logic port.

The dependency points INWARD always. The center never reaches out.

#### Real World Examples
- Enterprise banking systems
- Domain-driven design (DDD) implementations
- Systems that need to swap infrastructure (e.g. migrate from MySQL to PostgreSQL without touching business logic)
- Highly testable backend systems where tests use a "test adapter" instead of a real database

#### When to use
- Long-lived enterprise systems (10+ year lifespan)
- When testability is a top priority
- When you anticipate changing databases, UIs, or external services
- Domain-driven design projects

#### Strengths
- Business logic is completely isolated and testable
- Swap any adapter (database, UI, queue) without touching core
- Multiple adapters can exist simultaneously (REST + GraphQL + CLI)
- Forces clean thinking about what is "core" vs "infrastructure"

#### Weaknesses
- Significant upfront complexity and boilerplate
- Steep learning curve for teams new to the pattern
- Overkill for simple CRUD applications
- More files, more interfaces, more abstraction layers

---

## Category 2 — Distributed Patterns

> **Core question: How do I spread my system across multiple machines?**

These patterns define how to break a system into independently deployable and scalable pieces.

---

### 2.1 Service Oriented Architecture (SOA)

**Tag: Intermediate**

#### Definition
An architectural style where software components are designed as reusable services that communicate over a network through a common protocol. Services are coarse-grained and often share a common data layer.

#### How it looks
```
[ Service A ] ──┐
                │
[ Service B ] ──┼──▶ [ Enterprise Service Bus (ESB) ] ──▶ [ Shared DB ]
                │
[ Service C ] ──┘
```

#### SOA vs Microservices — The Key Difference

| | SOA | Microservices |
|---|---|---|
| Service size | Large, coarse-grained | Small, fine-grained |
| Communication | Enterprise Service Bus | Direct APIs or message queues |
| Data | Often shared database | Each service owns its data |
| Governance | Centralized | Decentralized |
| Era | 2000s enterprise | 2010s modern systems |

SOA is the **parent**. Microservices is the **evolution** — same idea, smaller services, no shared bus, no shared database.

#### Real World Examples
- Large enterprise systems (SAP, Oracle)
- Banking and insurance platforms
- Government IT systems
- Legacy enterprise applications

#### When to use
- Large enterprise with many existing systems to integrate
- When you need to expose business capabilities as reusable services
- Legacy modernization projects

#### Strengths
- Reusability of services across the enterprise
- Standardized communication protocols
- Better than monolith for large organizations

#### Weaknesses
- Enterprise Service Bus becomes a bottleneck and single point of failure
- Heavy governance and standards overhead
- Shared database creates coupling between services
- Slower to change than microservices

---

### 2.2 Microservices Architecture

**Tag: Intermediate**

#### Definition
An architectural style where a system is composed of small, independently deployable services. Each service owns its own data, runs in its own process, and communicates via APIs or message queues. Each service does exactly one business function.

#### How it looks
```
                    [ API Gateway ]
                          │
          ┌───────────────┼───────────────┐
          │               │               │
  [ User Service ] [ Order Service ] [ Payment Service ]
          │               │               │
      [ Users DB ]   [ Orders DB ]  [ Payments DB ]
```

#### The Core Principles
- **Single Responsibility** — each service does one thing
- **Own your data** — no shared databases between services
- **Independently deployable** — deploy one service without touching others
- **Failure isolation** — one service failing does not cascade
- **Technology freedom** — each service can use a different language or database

#### Real World Examples
- **Netflix** — 700+ microservices. Recommendation, streaming, billing, auth are all separate.
- **Uber** — trip management, driver matching, payment, notifications all separate services
- **Amazon** — the famous "two-pizza team" rule. If it takes more than two pizzas to feed the team, the service is too big.
- **Twitter** — tweet service, timeline service, notification service all independent

#### When to use
- Large teams that need to work independently
- When different features have very different scaling requirements
- When you need to deploy parts of the system multiple times per day
- When failure isolation between features is critical

#### When NOT to use
- Small teams (fewer than 15-20 engineers)
- Early stage product where requirements change constantly
- When you don't yet know where the service boundaries are

#### Strengths
- Independent scaling — scale only what needs scaling
- Independent deployment — ship the payment service without touching auth
- Team autonomy — each team owns their service end to end
- Fault isolation — payment service failing doesn't take down the whole app
- Technology diversity — use Python for ML service, Go for performance-critical service

#### Weaknesses
- Distributed systems complexity — network calls fail, latency increases
- Data consistency across services is hard — no single transaction spans multiple services
- Testing is harder — need to test service interactions
- Operational overhead — 50 services means 50 things to monitor, deploy, and debug
- Service discovery and communication infrastructure needed

#### The Microservices Death Star Problem
When every service calls every other service directly:
```
Service A ──▶ Service B ──▶ Service C
    │               │
    └──────▶ Service D ◀──────┘
```
This creates invisible coupling. The solution is event-driven communication (covered in Communication patterns).

---

### 2.3 Space-Based Architecture

**Tag: Advanced**

#### Definition
Removes the database as the central bottleneck by distributing both processing and data storage across multiple nodes in memory. All nodes share a common "tuple space" (in-memory data grid). Requests are handled by whichever node has capacity.

#### How it looks
```
[ User Request ]
       │
[ Load Balancer ]
       │
  ┌────┴────┐
  │         │
[ Node 1 ] [ Node 2 ]  ← Each node has full app + data in memory
  │         │
  └────┬────┘
       │
[ Async DB Write ]  ← Database updated asynchronously
```

#### Core Concept
Instead of all requests hitting one database, data lives in memory across all nodes. No single database bottleneck. Handles massive traffic spikes by adding nodes.

#### Real World Examples
- High-frequency trading platforms
- Ticket booking systems at peak load (concert tickets going on sale)
- Online gaming platforms
- Real-time bidding systems in advertising

#### When to use
- Extremely high concurrent user loads
- Systems with unpredictable massive traffic spikes
- When database is the proven bottleneck and caching isn't enough
- Real-time processing where milliseconds matter

#### Strengths
- Near-linear scalability — add nodes to handle more load
- No database bottleneck
- Extremely high throughput and low latency
- Handles massive spikes gracefully

#### Weaknesses
- Very complex to implement and operate
- Data consistency across nodes is extremely hard
- Expensive — keeping everything in memory costs money
- Not suitable for systems requiring strong data consistency
- Debugging distributed in-memory state is very difficult

---

### 2.4 Broker Architecture

**Tag: Intermediate**

#### Definition
A middleware component (the broker) coordinates communication between distributed services. Services do not communicate directly. They register with the broker, and the broker routes requests to the appropriate service, handles discovery, and manages communication.

#### How it looks
```
[ Service A ] ──▶┐              ┌──▶ [ Service C ]
                 │              │
[ Service B ] ──▶│   [ BROKER ] │──▶ [ Service D ]
                 │              │
[ Client ]   ──▶┘              └──▶ [ Service E ]
```

#### Broker vs Event-Driven
- **Broker** — the broker actively routes and coordinates. Services are known to the broker.
- **Event-Driven** — producers publish events. Consumers subscribe. Broker is passive message carrier.

#### Real World Examples
- **Apache Kafka** acting as message broker
- **RabbitMQ** routing messages between services
- **CORBA** (classic enterprise broker)
- **gRPC with service mesh** (modern broker pattern)

#### When to use
- When services need to discover each other dynamically
- When you need centralized routing and load balancing at the message level
- When you need protocol translation between services

#### Strengths
- Services are fully decoupled — they don't know each other's location
- Centralized routing logic
- Easy to add new services without changing existing ones
- Can handle protocol translation

#### Weaknesses
- Broker is a single point of failure if not made redundant
- Broker becomes a performance bottleneck at massive scale
- Adds latency to every service interaction
- Operational complexity of managing the broker

---

## Category 3 — Communication Patterns

> **Core question: How do the parts of my system talk to each other?**

---

### 3.1 Event-Driven Architecture

**Tag: Intermediate**

#### Definition
Components communicate exclusively through events. A producer publishes an event when something happens. Consumers subscribe to events they care about and react independently. No direct service-to-service calls.

#### How it looks
```
[ Order Service ]
       │
   "order.placed" event
       │
       ▼
[ Message Queue / Event Bus ]
       │
  ┌────┼────────────┐
  │    │            │
  ▼    ▼            ▼
[Payment] [Inventory] [Notification]
Service    Service      Service
```

#### The Three Components
- **Producer** — publishes events when something happens
- **Event Bus / Queue** — stores and routes events (Kafka, RabbitMQ, AWS SNS)
- **Consumer** — subscribes to specific events and reacts

#### CQRS — Lives Inside Event-Driven

**CQRS (Command Query Responsibility Segregation)** separates the write model (commands) from the read model (queries) completely. They are synced through events.

```
[ Client ]
    │
    ├──▶ COMMAND (write) ──▶ [ Write DB ] ──▶ event ──▶ [ Read DB ] ──▶ QUERY ──▶ [ Client ]
```

- **Command side** — optimized for writes. Enforces business rules. Normalized data.
- **Query side** — optimized for reads. Denormalized. Pre-computed views.
- **Sync** — an event triggers the read model to update when write model changes.

**Why CQRS lives here:** It fundamentally depends on events to sync the two sides. Without event-driven communication, CQRS cannot work.

**Real world:** Amazon product catalog (write and read are completely different data shapes), banking transaction systems.

#### Real World Examples of Event-Driven
- **Uber** — trip completed event triggers: payment processing + driver rating + rider notification + analytics — all simultaneously, all independently
- **Amazon** — order placed event triggers: inventory reservation + payment + shipping + email — each service reacts independently
- **Zomato / DoorDash** — order status events flow through the entire delivery pipeline

#### When to use
- When services need to be completely decoupled
- High volume async processing
- When one action triggers multiple downstream reactions
- Real-time data pipelines

#### Strengths
- Complete decoupling — producer does not know who consumes
- Scales independently — add consumers without touching producers
- Resilient — if consumer fails, event stays in queue and retries
- Easy to add new consumers without changing existing services
- Natural audit trail — every event is a record of what happened

#### Weaknesses
- Debugging is hard — tracing a request across multiple async consumers requires distributed tracing
- Event ordering is not guaranteed in all systems
- Eventually consistent — consumers process events at different speeds
- Complex error handling — what happens if payment consumer fails after inventory already reserved?
- Steep learning curve

---

### 3.2 Peer-to-Peer Architecture (P2P)

**Tag: Intermediate**

#### Definition
There is no central server. Every node (peer) in the network is both a client and a server simultaneously. Peers communicate directly with each other and share resources without any central coordinator.

#### How it looks
```
Traditional Client-Server:
[ Client ] ──▶ [ Server ] ◀── [ Client ]

Peer-to-Peer:
[ Node A ] ◀──▶ [ Node B ]
    │                │
    ▼                ▼
[ Node C ] ◀──▶ [ Node D ]
```

#### Characteristics
- No single point of control
- Each peer contributes resources (bandwidth, storage, compute)
- Network becomes more resilient as more peers join
- No central authority — governance is distributed

#### Real World Examples
- **BitTorrent** — file pieces distributed across all peers. No central file server.
- **Bitcoin / Blockchain** — no central bank. Every node validates transactions.
- **Ethereum** — smart contracts run on every node simultaneously
- **WebRTC** — video calls go peer to peer directly (Zoom uses P2P for small calls)
- **Early Napster** — music sharing directly between users (before it was shut down)

#### When to use
- Decentralized systems where no single entity should have control
- File sharing at massive scale
- Blockchain and cryptocurrency systems
- When resilience through distribution is the top priority

#### Strengths
- No single point of failure — network survives even if many nodes go down
- Scales naturally — more peers = more resources for the network
- No central server costs
- Censorship resistant — no single point to shut down

#### Weaknesses
- Security is extremely hard — no central authority to enforce rules
- Data consistency across nodes is a nightmare
- Performance is unpredictable — depends on peer availability
- Coordination problems — reaching consensus without a central authority requires complex algorithms (Raft, Paxos)

---

### 3.3 Pipe-Filter Architecture

**Tag: Intermediate**

#### Definition
Data flows through a sequence of processing steps (filters) connected by channels (pipes). Each filter receives data, transforms it, and passes it to the next filter. Filters are independent and know nothing about each other.

#### How it looks
```
[ Input ] ──▶ [ Filter 1 ] ──▶ [ Filter 2 ] ──▶ [ Filter 3 ] ──▶ [ Output ]
               (validate)       (transform)        (enrich)
```

#### The Two Components
- **Pipe** — the channel that carries data between filters. Passive connector.
- **Filter** — the processing step. Receives input, transforms it, passes output. Stateless.

#### Real World Examples
- **Unix command line** — `cat file.txt | grep "error" | sort | uniq` — each command is a filter
- **ETL pipelines** — Extract (filter) → Transform (filter) → Load (filter)
- **Video processing** — decode → resize → watermark → encode → upload
- **Compilers** — tokenize → parse → optimize → generate code
- **Spam filters** — receive email → check sender → check content → check attachments → deliver or reject
- **Apache Kafka Streams** — stream processing pipelines

#### When to use
- Data transformation workflows
- ETL (Extract, Transform, Load) processes
- Stream processing
- Any sequential data processing where steps are independent

#### Strengths
- Each filter is independent and reusable
- Easy to add, remove, or reorder filters
- Parallel processing — filters can run concurrently
- Easy to test each filter in isolation
- Natural fit for data transformation

#### Weaknesses
- Not suitable for interactive systems requiring low latency
- Data format must be agreed between filters
- Error handling between filters is complex
- Debugging requires tracing data through the whole pipeline

---

## Category 4 — Infrastructure Patterns

> **Core question: How does my system actually run and scale at the hardware level?**

---

### 4.1 Serverless Architecture

**Tag: Intermediate**

#### Definition
You write functions. The cloud provider runs them on demand. You do not provision, manage, or think about servers. Functions spin up when triggered, execute, and shut down. You pay only for the milliseconds they run.

#### How it looks
```
[ User Action / Event / API Call / Schedule ]
                    │
                    ▼
         [ Cloud Triggers Function ]
                    │
                    ▼
         [ Function Executes ]
                    │
                    ▼
         [ Function Shuts Down ]
         [ You pay for ~200ms ]
```

#### Characteristics
- **Stateless** — functions cannot hold state between invocations
- **Event-triggered** — HTTP request, schedule, queue message, file upload
- **Auto-scaling** — cloud runs 1 or 10,000 instances automatically
- **Pay per execution** — not paying for idle servers

#### Real World Examples
- **AWS Lambda** — most widely used serverless platform
- **Vercel** — deploys Next.js apps as serverless functions
- **Google Cloud Functions** — event-driven function execution
- **Coca-Cola** — vending machine transactions processed by Lambda
- **Nordstrom** — retail event processing
- **iRobot** — Roomba data processing pipeline

#### When to use
- Unpredictable or spiky traffic patterns
- Background processing tasks (image resize, email sending)
- APIs with low to medium traffic
- Startups that want zero infrastructure management
- Event-driven processing pipelines

#### When NOT to use
- Long-running processes (functions have max execution time limits)
- Applications requiring persistent connections (WebSockets need special handling)
- When cold start latency is unacceptable
- Very high frequency, consistent traffic (becomes expensive vs dedicated servers)

#### Strengths
- Zero server management — no patching, no provisioning
- Automatic scaling — handles traffic spikes without configuration
- Cost efficient for low/medium traffic — pay only for what runs
- Faster time to market — focus on code not infrastructure
- Built-in high availability from the cloud provider

#### Weaknesses
- **Cold start** — first invocation after idle period takes longer (100ms-2s)
- Vendor lock-in — AWS Lambda functions are not easily portable to Google Cloud
- Debugging is harder — no persistent server to SSH into
- Execution time limits — AWS Lambda max is 15 minutes
- Stateless — must use external storage for any state

---

### 4.2 Master-Slave Architecture

**Tag: Intermediate**

#### Definition
One node (master) handles all write operations and replicates data to multiple slave nodes. Slave nodes handle read operations. Reads and writes are separated at the infrastructure level.

#### How it looks
```
[ Application ]
      │
      ├──── WRITES ────▶ [ Master DB ]
      │                       │
      │                  replicates to
      │                  ┌────┴────┐
      │                  │         │
      └──── READS ─────▶ [ Slave ] [ Slave ]
                           DB 1     DB 2
```

#### Replication Types
- **Synchronous replication** — master waits for slave to confirm write. Slower but no data loss.
- **Asynchronous replication** — master does not wait. Faster but slaves may lag behind.

#### Replication Lag Problem
Because replication is often async, a user might:
1. Write a post (goes to master)
2. Immediately read it (goes to slave)
3. Slave hasn't received replication yet
4. User sees "post not found"

Solution: Route writes and immediate reads to master. Route all other reads to slaves.

#### Real World Examples
- **MySQL** replication — most common database setup for web applications
- **PostgreSQL** read replicas — standard setup for high-read applications
- **MongoDB** replica sets — primary (master) + secondaries (slaves)
- **Redis Sentinel** — master-slave with automatic failover

#### When to use
- Read-heavy applications (most web apps are 80% reads, 20% writes)
- When you need to scale reads without scaling writes
- Reporting and analytics queries that would slow down the main database
- Geographic distribution — put slaves close to users in different regions

#### Strengths
- Read performance scales linearly — add slaves to handle more reads
- Master is protected from read load
- Slaves can serve as hot standbys for disaster recovery
- Analytical queries run on slaves without affecting production writes

#### Weaknesses
- **Master is a single point of failure** for writes
- Replication lag causes eventual consistency issues
- Failover when master dies requires either manual intervention or orchestration (Sentinel, Patroni)
- Write bottleneck — all writes still go to one node
- Complexity of managing replication and monitoring lag

#### Master-Slave vs CQRS
| | Master-Slave | CQRS |
|---|---|---|
| Level | Infrastructure / Database | Application architecture |
| Separation | Read/write database instances | Read/write data models |
| Sync mechanism | Database replication | Domain events |
| Scope | Database scaling | Full application design pattern |

---

## Patterns That Don't Belong Here

This section exists because the internet incorrectly groups these with architectural patterns.

---

### MVC — Not a System Architecture Pattern

**What the internet says:** MVC is an architectural pattern.

**What it actually is:** MVC is an **application design pattern** that organizes code within a single application.

- **Model** = data and business rules
- **View** = user interface
- **Controller** = handles input and coordinates Model and View

MVC tells you how to organize your application code. It says nothing about how your system is distributed, how it scales, or how components communicate across a network.

**Where it belongs:** Inside Layered Architecture as an implementation of the presentation and business logic layers. It is layered thinking applied at the application code level — not a standalone architectural pattern.

**Analogy:** Saying MVC is an architectural pattern is like saying "alphabetical order" is a filing system. It is an organizing principle inside a filing system — not the system itself.

---

### Circuit Breaker — Not an Architectural Pattern

**What the internet says:** Circuit Breaker is an architectural pattern.

**What it actually is:** Circuit Breaker is a **failure resilience pattern** — it handles what happens when a downstream service is failing or slow.

```
[ Service A ] ──▶ [ Circuit Breaker ] ──▶ [ Service B ]
                        │
                  If Service B fails too often:
                  Circuit "opens" — stops calling B
                  Returns fallback response instead
                  Retries after timeout
```

It is about fault tolerance, not about how a system is structured. It belongs in **failure patterns** — which is a completely separate topic.

**Where it belongs:** Failure patterns, resilience patterns.

---

### Blackboard — Extremely Niche

**What it is:** A shared memory space (blackboard) where multiple specialized subsystems (knowledge sources) read and write to collaboratively solve a problem.

**Where it is used:** AI systems, expert systems, speech recognition, autonomous vehicles — extremely specialized domains.

**Why it doesn't belong in general system design:** 99% of engineers building web systems, APIs, or distributed systems will never encounter or need this pattern. It is not relevant to system design interviews or most real-world software engineering.

---

## Quick Reference Summary

| Pattern | Category | Core Question It Answers | Complexity |
|---|---|---|---|
| Monolithic | Structural | How do I start simple? | Beginner |
| Client-Server | Structural | How does a user talk to my system? | Beginner |
| Layered / N-Tier | Structural | How do I organize my code in layers? | Beginner |
| Microkernel | Structural | How do I make my system extensible? | Intermediate |
| Hexagonal | Structural | How do I protect my business logic? | Advanced |
| SOA | Distributed | How do I expose business services? | Intermediate |
| Microservices | Distributed | How do I scale teams and features independently? | Intermediate |
| Space-Based | Distributed | How do I eliminate database bottlenecks? | Advanced |
| Broker | Distributed | How do I decouple service discovery? | Intermediate |
| Event-Driven | Communication | How do services react without calling each other? | Intermediate |
| CQRS | Communication | How do I separate read and write models? | Advanced |
| Peer-to-Peer | Communication | How do I build with no central server? | Intermediate |
| Pipe-Filter | Communication | How do I process data in sequential stages? | Intermediate |
| Serverless | Infrastructure | How do I run code without managing servers? | Intermediate |
| Master-Slave | Infrastructure | How do I scale my database reads? | Intermediate |

---

## How These Patterns Relate to Each Other

Understanding relationships between patterns is what separates someone who memorized a list from someone who actually understands architecture.

```
Monolith ──grows──▶ Layered ──grows──▶ Microservices
                                              │
                                    naturally becomes
                                              │
                                       Event-Driven
                                              │
                                    CQRS lives inside

SOA ──evolved into──▶ Microservices

Client-Server ──is the base of──▶ Everything

Master-Slave ──scales data for──▶ Monolith or Microservices

Hexagonal ──can be applied inside──▶ Any pattern

Serverless ──can host──▶ Individual Microservices

Broker ──enables communication in──▶ Distributed systems

Pipe-Filter ──is often implemented with──▶ Event-Driven
```

---

## What Comes Next

Now that you understand architectural patterns — the categories, the hierarchy, the relationships, and what doesn't belong — you are ready for:

1. **HLD (High Level Design)** — How do you take these patterns and design a real system? What components do you pick? How do you draw the diagram?

2. **Building Blocks of HLD** — Load balancers, caches, CDNs, API gateways, message queues — the individual components you combine.

3. **Core Theoretical Concepts** — CAP theorem, consistency models, scalability, availability — the principles that guide your architectural decisions.

4. **Failure Patterns** — Circuit Breaker, Thundering Herd, Cascade Failure — how systems break and how to design against it.

5. **Real System Designs** — Design Twitter. Design Uber. Design YouTube. Apply everything above.

---

*All content original. Created by @kriti.sde*
