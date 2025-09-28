## 1. Overview

Many organizations struggle with managing secure authentication across multiple platforms while balancing user data ownership, compliance, and seamless integration with third-party applications. Traditional solutions often create trade-offs between security, flexibility, and ease of adoption, leaving gaps in reliability, multifactor support, and centralized user management.

To address this, AuthBox delivers a modular, standards-based architecture that supports OIDC, while offering deployment flexibility as either a system of record or a broker to client-managed stores. It includes encrypted data storage, asynchronous email services for MFA and password resets, and an admin dashboard for user and role management. Designed for replication per client via IaC, this IdP ensures isolation, compliance, and scalability, making it secure and extensible for modern enterprise needs.

## 2. Requirements

### 2.1 Functional Requirements

1. To act as an IdP.
2. To authenticate user from the expected platform.
3. To allow integrations to software products.
4. To interact with client's authentication APIs without reading/writing data stores and get authentication tokens when client chooses to store data on their end.
5. To store data in an encrypted format if authentication is on the service end.
6. To provide multifactor authentication, if enabled by the clients.
7. To provide password update functionality to the users.
8. To provide admin dashboard to client admins to manage their user base.
9. To isolate systems based on client’s tier:
    1. Free tier clients are part of the same ecosystem.
    2. Standard tier clients are part of the shared ecosystem, which is distributed over multiple stacks.
    3. Enterprise tier clients are part of an isolated ecosystem.
10. To should store just minimal data which is required for authentication and authorization.
11. To manage Role Based Access Control (RBAC), like LDAP, POSIX, etc., for clients to handle resource access.

### 2.2 Non-Functional Requirements

1. The service should be highly available, i.e., 99.9%.
2. The service should have a lower latency with a p99 of less than or equal to 300ms.
3. The service should be able to support thousands of concurrent requests per client.
4. The service should be able to scale automatically based on the traffic.
5. The service should be compliant to Swiss Federal Act of Data Protection (FADP) and EU General Data Protection Regulation (GDPR) policies.
6. The service should store data in an encrypted & non-accessible format.
7. The service should have high observability, i.e., monitoring, alerting and logging.
8. The error rates within the service should be very low, with a p99 of less than or equals to 0.01%.

## 3. System Architecture

![Screenshot From 2025-09-07 16-42-12.png](https://github.com/user-attachments/assets/bbc38da7-5053-487f-8c3f-934d5ae133e5)

The system is a multi-tenant Identity Provider (IdP) stack designed for scalability, security, and compliance. User requests flow through a CDN into an API Gateway/Proxy, which manages routing, throttling, and security enforcement. The backend consists of FaaS components for Authentication, Role Management, and User Management.

- Authentication FaaS handles token issuance (OIDC-compliant), credential validation, and integration with client-side auth APIs without persisting client data.
- Role Management FaaS interacts with a dedicated Role Database and leverages Redis for caching policies/permissions to minimize latency.
- User Management FaaS manages user lifecycle operations and writes to the User Database with full encryption-at-rest and in-transit guarantees.

An Emailing Service operates in a separate VPC, consuming messages asynchronously from the system (via API Gateway) and storing templates/logs in a storage bucket before dispatching through an SMTP/transactional mail provider.

The architecture is instrumented with Prometheus and Grafana for metrics, monitoring, and alerting. Client services integrate via OIDC or custom APIs, depending on configuration. System isolation is enforced based on tiers (free, standard, enterprise) with different deployment footprints.

This setup prioritizes low latency (p99 ≤ 300ms), high availability (99.9%), compliance (FADP/GDPR), and auto-scaling while keeping the control plane lightweight and extensible.

## 4. Data Architecture

### 4.1 Data Requirements

1. We need to store data from a small to large volumes of data based on client requirements for both user and role databases.
2. Our system would require a read heavy database.
3. Each object we store for user database would be small (like 100 KBs).
4. Each object we store for role database would be small (like 50 KBs).
5. To store email templates, a BLOB storage would be required like S3 or Firebase (each template would be around 1MB on average).
6. Since, our system is read-heavy, a caching system would be required to reduce loads on both user and role databases.
7. The system requires lower latency to enhance authentication experience.
8. No complex relations are required for any of the data stores.

### 4.2 Database Type Comparison

- SQL
    - Pros:
        - Strong ACID guarantees.
        - Good for structured relationships (e.g., users ↔ roles).
        - Mature ecosystem, rich query support (joins, aggregations).
        - Easy integration with ORM layers.
        - Fast if data is indexed and fits in memory.
        - Can scale with read ****replicas and caching.
        - Slower for massive scale, since it enforces ACID + indexes + relationships.
    - Cons:
        - Scaling horizontally is harder (sharding).
        - Might struggle with very high read volumes if not cached.
        - Less flexible for evolving schema (but Postgres JSONB helps).
- NoSQL (Document)
    - Pros:
        - Built for scalability and high throughput.
        - Flexible schema for evolving user/role attributes.
        - Great for read-heavy workloads when combined with secondary indexes.
        - DynamoDB/MongoDB handle auto-sharding and horizontal scaling easily.
        - Optimized for point lookups / key-based reads.
        - MongoDB and DynamoDB can do sub-5 ms reads at scale.
        - Generally faster writes (eventually consistent in some cases).
    - Cons:
        - Weaker transactional guarantees (though newer versions have improved).
        - Joins are harder (but users ↔ roles might not be deeply relational, so manageable).
        - Query flexibility limited compared to SQL.

### 4.3 Conclusion on data stores

- Based on the above comparison, NoSQL DB like DynamoDB would be a good fit for both user and role databases.
- For BLOBs apps like S3 or Supabase buckets would be a good choice. Since, there are minimal files, Supabase would be a good choice.
- Having a cache DB like Redis would be a good choice to decrease latency even further.

## 5. Component Design (WIP)

***WIP***

## 6. Tech Stack

### 6.1 Frontend

- **Language:** TypeScript (TS) would be the best choice as it provides data type binding to objects, reducing the risk of getting/sending invalid data objects to and fro the server. JavaScript (JS) would also be a suitable language but comes with a huge number of uncertainties due to being an open-ended language.
- **Styling:** Tailwind with Sass would be a better choice when compared to using CSS directly, as Tailwind provides in-built functionalities handling common CSS caveats. Sass helps in better handling nested HTML classes rather than using normal CSS where the tree path has to be defined for each use case. There are other alternatives to Tailwind like bootstrap, Material UI, etc.
- **Framework:** Vue would be a good choice of framework due to its simplicity and 2-way data binding with state management, making it highly flexible for our use-case. Other options are:
    - React
    - Angular
- **Content Delivery Network (CDN):** A CDN would be required for faster loading of UI, where it increases the speed to load the images and the components. Cloudfare would be a good choice for our use case as it has in-built capabilities to resist DDoS and provides other security features which improves application security. Other than security, it provides global coverage and has easy to on-board setup. Other viable options are:
    - Vercel
    - Netlify
    - Fastly
    - Akamai
    - AWS CloudFront

### 6.2 Backend

- **Communication Pattern:** Remote Procedure Calls (RPC) would be a perfect choice as they have defined request-response models with running on HTTP2 which a light-weight and faster version of HTTP1.1 which is being currently used in REST/SOAP communications. Since, request and response models are well-defined, it reduces the risk of receiving or getting unknown object structures and also doesn’t require us to define a separate interface package for data transfer objects.
- **Language:** Both java and golang would be great options but golang takes an edge due to its lightweight nature, instant start time, high performance and low latency.
- **Interface Definition Language (IDL):** Protobuf is a mature IDL providing both native RPC and REST support (using gRPC Gateway). Other than these protocols support, it is compatible with a high number of languages. There are other IDLs like Smithy, Spring RPC and Thrift which are a bit less mature and might not be compatible with other languages.

### 6.3 Compute

- Function as a Service (FaaS) would be a better option compared to Infrastructure as a service (IaaS) as we don’t require to store any details in the local cache of the system, expecting the service to just be available only when required, i.e., only when a request is made.
    - AWS Lambda is a perfect fit for the use case as it is fully managed, auto-scalable and provides AWS layered security. Though it has an overhead of cold starts, being a lightweight application the duration should be as minimal as ~200ms.
    - OpenFaaS is also a good choice, which is K8s self-managed system. Provides an opportunity for no cold start as it allows having at least 1 instance running all the time. Since, being a K8 based service, it requires huge operational overhead and requires additional policy management for peak scalings. Also, it’s a flat-rate service, as it depends on the number of underlying nodes available at a point. The initial setup also requires intensive knowledge about underlying resources, followed by maintainability.
    - Other viable options:
        - Netlify Functions
        - Cloudfare Workers
        - Kubeless
        - KNative
        - OpenWisk
        - Google Cloud Functions
        - Azure Functions

## 7. Security

All sensitive operations are processed within isolated VPC segments, ensuring network-level protection for authentication, user, and role management functions. Managed identity services such as OIDC and client APIs provide robust standards-based authentication, eliminating common vulnerabilities. API traffic is filtered and routed via a centralized API Gateway/Proxy, enabling unified policy enforcement, rate limiting, and access logging. Each Function-as-a-Service (FaaS) executes with least-privilege roles when interacting with user and role databases, while inter-service communications leverage secure channels. The emailing service operates within its own VPC, using restricted storage and access controls. This architecture minimizes attack surfaces and supports comprehensive auditing, protecting user data and workflow integrity throughout the system

## 8. Observability & Reliability

For the suggested architecture, we would go ahead with a hybrid model for monitoring. Metrics for performance and health will rely on CloudWatch Metrics, automatically generated by AWS services like Lambda, API Gateway, and the CDN. For logs, application-level logging from the FaaS (Lambda) functions will be redirected from CloudWatch Logs to a centralized Grafana-Loki stack to avoid CloudWatch ingestion costs. This will be achieved by direct logging using telemetry APIs provided by Loki to intercept the logs. Finally, reliability will be addressed by configuring Grafana. Grafana's alerting is highly flexible, supporting various communication tools like email, Slack, PagerDuty, Webhooks, and more, all managed from a single place.

## 9. Conclusion

The proposed AuthBox architecture delivers a secure, scalable, and highly available identity management solution tailored for modern enterprise needs. By leveraging a modular FaaS backend, gRPC-based communication, and NoSQL databases with caching layers, the system achieves low-latency authentication, efficient role management, and extensible user lifecycle operations. Tiered isolation ensures compliance and security for varying client requirements, while observability via Prometheus, Grafana, and centralized logging guarantees operational reliability. The design balances performance, maintainability, and flexibility, providing a foundation for future feature growth and multi-cloud deployment without compromising data security or user experience.
