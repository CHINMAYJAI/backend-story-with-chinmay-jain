## Backend Scaling and Performance Engineering: Part-1
_1. Define and Understand Performance Metrics_
Before making changes, you must establish how to accurately measure system speed and capacity.

- _Measure Latency in Percentiles:_ Do not use averages to measure latency, as they hide the worst experiences of your users. Instead, use percentiles like P50, P90, and P99. A P99 of 2 seconds, for example, means 99% of requests are faster than 2 seconds, while the slowest 1% experience 2-second delays. These higher percentiles often represent your most valuable customers making complex transactions, like purchases.
- _Evaluate Throughput:_ Throughput measures how many requests your system handles in a given time period, such as requests per second. Throughput and latency are interconnected; as throughput reaches high levels, latency will eventually spike.
- _Manage Utilization:_ System utilization represents the percentage of a system's computational capacity actively in use. As servers approach 100% utilization, requests form a queue and wait times grow exponentially, not linearly. Always run systems between 60% and 80% utilization to leave a headroom buffer for unpredictable bursts of traffic.

_2. Identify the Exact Bottleneck_
Do not guess what is slowing down your system or blindly add solutions like caching; always measure first to find the specific bottleneck.

- _Use Distributed Tracing:_ Because most backend performance issues are I/O bound (like database queries or external API calls), distributed tracing is used to follow a request end-to-end to see exactly how many milliseconds each system component added.
- _Use Profilers and Flame Graphs:_ For CPU-bound tasks (like heavy JSON serialization), profilers record what happens during runtime. Flame graphs visually represent this call stack to help you instantly spot which functions are taking up the most compute time.

_3. Optimize the Database Layer_
Databases are typically the primary bottleneck because they do the hard work of storing data to disk durably and handling concurrent operations.

- _Fix N+1 Queries:_ This happens when you make one database query to fetch a list of items, and then run a loop to make a separate database query for each individual item's details. Instead, fetch data in bulk using SQL Joins or Object-Relational Mapping (ORM) primitives (like `select_related` or `includes`) to retrieve everything in just two queries.
- _Implement Indexes:_ Indexes maintain a sorted copy of a column (using a B-tree data structure) so the database can locate rows instantly rather than performing a slow "sequential scan" over millions of rows. However, indexes take up storage space and slow down write operations (inserts, updates, deletes) because the index data structure must be updated concurrently. You can verify if an index is actively being used by running the `EXPLAIN ANALYZE` command.
- _Set Up Connection Pooling:_ Setting up a TCP connection for every query is expensive and will easily exhaust a database's hard limit for concurrent connections during traffic spikes. Connection pools maintain a set of open connections that servers can quickly borrow and return. Use external poolers (like PgBouncer) when horizontally scaling so that multiple auto-scaled server instances do not independently overwhelm the database's connection limit.

_4. Implement Caching_
Caching stores the results of expensive operations in fast memory to reduce perceived latency drastically.

- _Choose a Caching Pattern:_ The "Cache Aside" pattern checks the cache first, falling back to query the database if the data is missing. "Write Through" updates the cache synchronously alongside database writes to prevent cache misses. "Write Behind" updates the cache instantly for a fast user response while updating the database asynchronously.
- _Design Cache Invalidation:_ Keeping cached data from becoming stale is notoriously difficult. You can use Time-based invalidation (TTL) to automatically expire data after a set period, or Event-based invalidation to explicitly delete the cache entry precisely when the underlying database record is updated.
- _Consider Tiered Caching:_ You can combine local caching (in-memory, lightning-fast but prone to inconsistency across different servers) with distributed caching (like Redis, which ensures consistency but adds minor network latency).

_5. Scale the Servers_
When software optimizations are exhausted and traffic continues to grow, you must add hardware capacity.

- _Vertical Scaling (Scaling Up):_ This involves upgrading your existing single server with more CPU cores, RAM, or faster SSDs. It requires zero architectural changes but has hard hardware limits, creates a single point of failure (if the server crashes, your service is down), and lacks geographic distribution for users far from the server.
- _Horizontal Scaling (Scaling Out):_ This involves adding multiple identical server instances that work together to handle traffic. It offers effectively infinite capacity, fault redundancy, and geographic distribution. However, it introduces immense architectural complexity, requiring load balancers and difficult distributed state management.

## Backend Scaling and Performance Engineering: Part-2

_1. The Requirement of Statelessness for Horizontal Scaling_
Horizontal scaling involves adding multiple identical server instances to handle traffic. However, this is only possible if the system is **stateless**, meaning no single server instance holds exclusive data in its local memory. If a user authenticates with Server A, and the next request goes to Server B, Server B will throw an error if the session data was only stored locally on Server A. To solve this, **all state must be externalized**: session data should go to an in-memory database like Redis, file uploads should go to object storage like S3 or Cloudflare R2, and relational data must go to a centralized database like PostgreSQL.

_2. Load Balancers_
A load balancer is a mandatory middleman in a horizontally scaled architecture that decides which server instance should process a user's request.

- _Algorithms:_ The simplest logic is _Round Robin**, which sends requests in a rotating order. However, if some requests are heavy (e.g., 2 seconds) and others are light (e.g., 200ms), a mindless rotation can crash a server by accidentally sending it all the heavy requests. A smarter approach is the **Least Connections_ algorithm, which routes new requests to the server with the fewest currently active connections, naturally balancing heavy and light workloads.
- _Health Checks:_ To prevent sending traffic to a crashed server, the load balancer sends test requests (e.g., every second) to all servers. If a server fails to return a 200 success response, the load balancer immediately blacklists it and stops routing user traffic to it until it recovers.

_3. Database Scaling Strategies_
Because databases hold state, they cannot just be mindlessly duplicated like application servers.

- _Read Replicas:_ Since 70% to 90% of backend traffic typically consists of read operations, you can set up one "primary" database instance exclusively for write operations, and multiple "read replicas" spread across the globe purely for select queries. The major trade-off is **replication lag**—the physical time (e.g., 200ms) it takes for data to travel from the primary database to the replicas. This consistency issue can cause a user to update their profile and immediately see old data.
- _Sharding (Partitioning):_ Sharding involves physically dividing a massive table (like billions of rows) across multiple different database instances using a "sharding key," such as dividing orders by date. This instantly reduces query latency and increases capacity.
- _Distributed Databases:_ Setting up replication and sharding manually is highly complex. Modern architectures rely on managed distributed databases (like PlanetScale, Neon, or CockroachDB) that handle these deep infrastructure problems automatically.

_4. Content Delivery Networks (CDNs) and Edge Computing_
Data transmission is capped by the speed of light; a request round-trip from Tokyo to an East Coast US server takes an absolute minimum of 100 milliseconds, plus the time required to deserialize JSON, run business logic, and query the database.

- _CDNs:_ To bypass this, CDNs cache static content (HTML, JS bundles, images) and specific API responses on "edge nodes" located geographically close to the users. This drops latency from 100+ milliseconds to just 2-3 milliseconds and significantly reduces the load on the primary servers. CDNs like Cloudflare also act as a security shield, absorbing massive DDoS attacks.
- _Edge Computing:_ Instead of just serving static files, edge computing involves running lightweight code directly at the CDN layer. For example, using Cloudflare Workers (which use ultra-fast V8 isolates), you can validate a user's session token at the edge node. If unauthorized, the edge node immediately returns a 401 error, sparing your primary server from processing the bad request. Edge nodes have strict constraints, however, and lack robust RAM, CPU, or file system access.

_5. Asynchronous Processing_
To reduce perceived latency for users, offload non-critical tasks to the background. If a user triggers a workflow that takes 400ms (100ms for database updates + 300ms to ping an external email provider), do not make the user stare at a loading spinner. Instead, update the database, instantly return a success response to the user, and push the task of sending the email into a _Message Queue_ (like RabbitMQ or a Redis queue). A background worker will eventually pick up the task and process it. This is ideal for sending emails, video uploads, image resizing, or deleting bulk account data.

_6. Microservices vs. Monoliths_
A monolith is a single deployable unit containing all application logic, making it simple to develop, test, and deploy. **Microservices exist primarily to scale the human team, not machine performance**. When teams grow past 100+ developers, monoliths suffer from deployment dependencies (one team's broken code blocking another's release). Splitting an application into independent microservices allows teams to deploy independently, scale resource-heavy services separately, and even use different programming languages for specific tasks (e.g., using Rust for image processing and Node.js for markdown parsing). However, microservices introduce massive complexities: pure function calls become slow network calls, debugging requires complex distributed tracing, and maintaining data consistency across multiple independent databases is notoriously difficult.

_7. Serverless Computing_
Traditional servers require **capacity planning**. If you over-provision, you waste thousands of dollars paying for unused idle capacity. If you under-provision, your app crashes during traffic spikes. Even traditional autoscaling is fundamentally reactive and takes minutes to boot up new operating systems.

- _The Serverless Model:_ You only provide the code (functions) and the routing events. The provider (like AWS Lambda or Vercel) dynamically spins up the necessary computing power the exact moment a request arrives, and you **only pay for the exact milliseconds of execution time**, saving vast amounts of money.
- _The Drawbacks:_ Serverless architectures suffer from **Cold Starts**—the latency penalty incurred when the provider has to boot up a fresh container and language runtime to handle a new request. They also mandate complete statelessness (no long-lived TCP or WebSocket connections) and enforce strict execution time limits.

_8. Core Rules of Performance Engineering_

- _Measure before optimizing:_ Never guess the bottleneck. Use observability tools (logs, metrics, distributed tracing via Prometheus/Grafana or New Relic) to find out exactly where time is being spent before implementing solutions.
- _Prefer simple solutions:_ Choose a larger server over horizontal scaling, or proper database indexing over implementing a Redis cache, unless simplicity has genuinely failed you. Complexity is a massive operational cost.
- _Scale for your actual problems:_ Do not prematurely architect a system for a million users on day one. Build for your current scale with a reasonable buffer.
- _Implement observability on day one:_ Measuring your system's health prevents random crashes and eliminates guessing.
