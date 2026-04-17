# AWS Principal Architect — Deep Dive Interview Reference
> **Level:** Senior → Staff / Principal Engineer  
> **Depth:** Internal implementation, not documentation  
> **Author tone:** "How does it work internally?" after every sentence

---

## TABLE OF CONTENTS

1. [AWS Lambda](#1-aws-lambda)
2. [API Gateway](#2-api-gateway)
3. [Amazon S3](#3-amazon-s3)
4. [Amazon DynamoDB](#4-amazon-dynamodb)
5. [Amazon SQS](#5-amazon-sqs)
6. [Amazon SNS](#6-amazon-sns)
7. [AWS Step Functions](#7-aws-step-functions)
8. [Amazon CloudFront](#8-amazon-cloudfront)
9. [Application Load Balancer (ALB)](#9-application-load-balancer-alb)
10. [Network Load Balancer (NLB)](#10-network-load-balancer-nlb)
11. [Amazon VPC](#11-amazon-vpc)
12. [Amazon ECS](#12-amazon-ecs)
13. [AWS Fargate](#13-aws-fargate)
14. [Amazon EC2](#14-amazon-ec2)
15. [Amazon CloudWatch](#15-amazon-cloudwatch)
16. [AWS X-Ray](#16-aws-x-ray)
17. [SPECIAL DEEP DIVES](#17-special-mandatory-deep-dives)
    - [Lambda + SQS Internal Flow](#171-lambda--sqs-event-source-mapping-internal-flow)
    - [API Gateway → Lambda Flow](#172-api-gateway--lambda-internal-flow)
    - [DynamoDB Internals](#173-dynamodb-deep-internals)
    - [CloudFront Internals](#174-cloudfront-deep-internals)
    - [Step Functions Execution](#175-step-functions-internal-execution)
    - [ECS vs Fargate vs EC2](#176-ecs-vs-fargate-vs-ec2-real-differences)
    - [IAM Deep Dive](#177-iam-deep-dive)

---

# 1. AWS LAMBDA

## 1.1 Core Purpose

**Exact distributed systems problem solved:**  
Lambda solves the problem of **stateless compute elasticity** — provisioning, managing, and scaling isolated compute units on-demand without any persistent server state. It eliminates the O(n) operational overhead of managing a fleet of EC2 instances when compute demand is bursty or unpredictable.

**What breaks without it:**  
You'd need to maintain a constantly-running server pool, over-provisioned for peak load, or build your own auto-scaling orchestration layer (ECS/K8s). Bursty workloads would require either wasted idle capacity or significant lag during scale-out.

---

## 1.2 Internal Architecture

### Control Plane vs Data Plane

```
CONTROL PLANE (Management)                DATA PLANE (Invocation)
─────────────────────────────             ──────────────────────────
- Lambda Console / API                    - Invoke API endpoint
- Function CRUD (CreateFunction,          - Frontend Invoke Routers
  UpdateFunctionCode)                     - Worker Manager
- Layer management                        - Sandbox environment (MicroVM)
- IAM permission evaluation               - Execution environment
- VPC attachment (ENI provisioning)       - Log shipping to CloudWatch
```

### Internal Components

```
                        ┌─────────────────────────────────────────┐
                        │           LAMBDA CONTROL PLANE           │
                        │  ┌─────────┐  ┌──────────┐  ┌───────┐  │
                        │  │Function │  │  Layer   │  │  IAM  │  │
                        │  │Registry │  │  Store   │  │Svc    │  │
                        │  └─────────┘  └──────────┘  └───────┘  │
                        └─────────────────────────────────────────┘
                                          │
                         ┌────────────────┼───────────────────┐
                         │                │                   │
                ┌────────▼───────┐  ┌─────▼──────┐  ┌───────▼──────┐
                │ Frontend Load  │  │  Worker    │  │  Placement   │
                │   Balancers    │  │  Manager   │  │   Service    │
                │ (Invoke Router)│  │            │  │              │
                └────────┬───────┘  └─────┬──────┘  └───────┬──────┘
                         │                │                  │
                         └────────────────▼──────────────────┘
                                    ┌──────────┐
                                    │  Worker  │
                                    │  Fleet   │
                                    │          │
                                    │ ┌──────┐ │
                                    │ │MicroVM│ │  ← Firecracker VMM
                                    │ │(Slot) │ │
                                    │ └──────┘ │
                                    └──────────┘
```

### Firecracker MicroVM — The Real Isolation Unit

Lambda does NOT use Docker containers directly. AWS built **Firecracker**, a KVM-based microVM manager written in Rust.  

Each execution environment = 1 Firecracker MicroVM:
- Uses KVM for hardware virtualization (not emulation)
- 125ms boot time for a new MicroVM (this IS the cold start)
- Each MicroVM has its own kernel, network namespace, cgroups
- Multi-tenancy is achieved by running multiple MicroVMs on the same physical host, isolated via hardware VM boundaries
- MicroVM memory is snapshotted for warm starts (the runtime and init code are already loaded)

### Request Lifecycle (Step by Step)

```
1. Client → Invoke API (HTTPS POST to lambda.us-east-1.amazonaws.com)
2. Frontend Router (global ELB layer) terminates TLS, authenticates
3. Auth: SigV4 signature verified → IAM policy evaluated (GetPolicy, simulate)
4. Frontend routes to correct Regional Worker Manager
5. Worker Manager checks: is there a WARM sandbox available?
   YES → assigns request to warm sandbox, skips init
    NO → Placement Service picks a physical Worker host
        → Worker Manager sends "Create Sandbox" to that Worker
        → Firecracker spawns MicroVM (allocates memory, vCPU slots)
        → Runtime bootstrap: download function code from S3 (zipped, AES-256 encrypted)
        → Runtime init (Node.js: v8 engine starts, runtime layer loads, handler module loaded)
        → INIT phase runs (module-level code, DB connection pools, etc.)
        → Runtime signals "ready" to Worker Manager via internal pipe
6. Worker Manager delivers event payload to sandbox
7. Runtime invokes handler function
8. Response OR error returned via internal IPC
9. Worker Manager streams logs to CloudWatch Logs (via log agent in MicroVM)
10. Sandbox state: either WARM (kept for reuse) or RECYCLED (if OOM, crash, or TTL exceeded)
```

### How Multi-Tenancy Isolation Works

- **Hardware VM isolation:** Each MicroVM is a full VM with its own kernel. A neighbor tenant cannot read your memory even with speculative execution attacks (Meltdown/Spectre mitigated by VM boundary).
- **Network isolation:** Each MicroVM gets its own network namespace. Outbound traffic goes through NAT on the Worker host, not shared with other tenants.
- **CPU:** cgroups limit CPU shares. vCPU is mapped to a host thread. With 1,769MB RAM = 1 full vCPU equivalent.
- **Storage:** Each MicroVM gets a rootfs + read-only layer from the function ZIP (mounted via overlay fs).

### Scaling Mechanics

Lambda scales by **burst concurrency**, not by traditional auto-scaling:

```
Burst limit per region: 3,000 concurrent executions initially
Then: +500/minute until account concurrency limit (default 1,000 unreserved)

Internally:
- Worker Manager tracks "available warm slots" per function ARN
- When demand > available slots, Placement Service provisions new Workers
- New Workers can boot Firecracker in ~125ms
- Each physical Worker host can run MULTIPLE MicroVMs (multi-tenant packing)
- Reserved concurrency = hard cap on number of simultaneous live MicroVMs for that function
```

### Failure Handling Internally

- **Sync invocation (RequestResponse):** Error propagates to caller. No retry by Lambda. Client must retry.
- **Async invocation (Event type):** Lambda has internal event queue. Retries 2x with exponential backoff (~1min, ~2min). Then routes to DLQ/EventBridge if configured.
- **Worker crash:** Worker Manager detects missing heartbeat from MicroVM → marks slot unavailable → provisions replacement. In-flight request may get a 500.
- **Sandbox OOM:** Kernel sends SIGKILL to MicroVM process → Worker Manager cleans up → function logs report "Runtime exited with error: signal: killed"

### Background Services / Hidden Components

- **Poller Service (for Event Source Mappings):** Separate fleet of pollers for SQS, Kinesis, DynamoDB Streams — NOT Lambda workers themselves (covered in Section 17.1)
- **Log Agent:** Inside each MicroVM, a sidecar process ships `/dev/log` to CloudWatch Logs asynchronously
- **ENI Manager:** For VPC Lambdas, a hyperplane ENI is provisioned; modern Lambda uses "hyperplane" model where ENIs are shared across MicroVMs via internal routing (eliminated the 90-second VPC cold start)
- **Code Optimization Service:** Asynchronously compiles/optimizes your function ZIP, stores optimized image in S3 for faster cold starts

---

## 1.3 Invocation & Communication Model

| Mode | Who Initiates | Sync/Async | Notes |
|------|--------------|------------|-------|
| RequestResponse | Client | Sync | Client waits, 900s max |
| Event | AWS service (S3, SNS) | Async | Lambda retries 2x |
| DryRun | Client | Sync | Validates perms, no execution |
| ESM (SQS/Kinesis) | Lambda Poller | Pull | Batch processing |

**Backpressure:**  
- Async queue: Lambda throttles and buffers up to 6 hours
- ESM: Visibility timeout / iterator position controls backpressure (no new batches until current processed)
- Reserved concurrency = 0 → effective circuit breaker

---

## 1.4 Deep-Dive Scenario (Node.js)

```
HIGH-SCALE ORDER PROCESSING API

Client
  │
  ▼
API Gateway (HTTP API) ──────────────────────────────────────┐
  │                                                          │
  ▼                                                          │
Lambda: order-validator                                      │
  │  (sync, <50ms, validates schema, checks inventory)       │
  │                                                          │
  ├──→ DynamoDB: write order record (status=PENDING)         │
  │                                                          │
  └──→ SQS: enqueue fulfillment event                        │
             │                                               │
             ▼                                               │
         Lambda: fulfillment-processor (ESM, batch=10)       │
           │  (async, calls payment service, inventory)      │
           └──→ SNS: notify downstream services              │
                  │                                          │
                  ├──→ Lambda: email-notification            │
                  └──→ Lambda: analytics-ingester            │
```

```javascript
// order-validator/index.js
const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');

// INIT PHASE: runs once per cold start, OUTSIDE handler
// These clients are reused across invocations (connection pooling)
const dynamo = new DynamoDBClient({ region: process.env.AWS_REGION });
const sqs = new SQSClient({ region: process.env.AWS_REGION });

exports.handler = async (event) => {
  // HANDLER: runs per invocation
  const body = JSON.parse(event.body);
  
  // Write to DynamoDB
  await dynamo.send(new PutItemCommand({
    TableName: 'Orders',
    Item: {
      orderId: { S: body.orderId },
      status: { S: 'PENDING' },
      timestamp: { N: String(Date.now()) }
    },
    ConditionExpression: 'attribute_not_exists(orderId)' // idempotency
  }));

  // Enqueue for async processing
  await sqs.send(new SendMessageCommand({
    QueueUrl: process.env.FULFILLMENT_QUEUE_URL,
    MessageBody: JSON.stringify(body),
    MessageGroupId: body.customerId, // FIFO ordering per customer
    MessageDeduplicationId: body.orderId // idempotency
  }));

  return { statusCode: 202, body: JSON.stringify({ orderId: body.orderId }) };
};
```

**Key architectural decisions:**
- Clients initialized at module-level → reused across warm invocations
- Async flow via SQS → decouples validator latency from fulfillment latency
- DynamoDB ConditionExpression → idempotent (safe Lambda retry)

---

## 1.5 When to Use (With Tradeoffs)

| Use When | Tradeoff Accepted |
|----------|------------------|
| Bursty, unpredictable traffic | Cold start latency (50ms–2s depending on runtime + VPC) |
| Event-driven, stateless compute | No persistent connections (workaround: init-phase pooling) |
| Per-request billing matters | Higher per-unit cost vs EC2 at sustained high throughput |
| Short-lived jobs <15min | Can't run long-running batch processing |
| Parallel fan-out (map-reduce pattern) | 1,000 concurrency limit by default, burst limits apply |

---

## 1.6 When NOT to Use

| Anti-Pattern | Failure Scenario |
|--------------|-----------------|
| Long-running ML inference (>15min) | Function times out, partial work lost |
| Stateful workflows (session state in memory) | Different invocations hit different sandboxes — state lost |
| High-throughput sustained load (>10k RPS constant) | Cost 3-5x higher than EC2/ECS at sustained load |
| Sub-millisecond latency requirements | Cold starts make P99 latency unpredictable |
| Large binary/file processing (>512MB in /tmp) | Ephemeral storage limit; use EFS instead |
| Connection-heavy workloads (many DB connections) | Each Lambda = new connection; use RDS Proxy |

---

## 1.7 Common Pitfalls (Real Production Issues)

1. **Keeping mutable state in global scope:** Global arrays/maps persist across warm invocations of the SAME sandbox — appears to work, but two concurrent requests hit different sandboxes and see different state. Creates ghost bugs.

2. **Not handling partial batch failures (SQS ESM):** Without `ReportBatchItemFailures`, a single failed message in a batch of 10 causes ALL 10 to be requeued → repeated poison pill processing → queue backup.

3. **VPC Lambda cold start (legacy):** Pre-hyperplane, each cold start allocated a new ENI (~45-90 seconds). Caused thundering herd during scale-out. **Fix:** Use hyperplane model (default for new functions), or keep functions warm via EventBridge scheduled ping.

4. **Environment variable secrets:** Storing DB passwords in env vars. They show up in CloudTrail and Lambda console. **Fix:** SSM Parameter Store or Secrets Manager with SDK call in init phase.

5. **Sync invocation chains:** Lambda A → Lambda B → Lambda C synchronously. Each adds timeout risk and you pay for idle wait time. **Fix:** Use Step Functions or async SQS queuing between steps.

6. **Not setting reserved concurrency:** Noisy neighbor Lambda consumes all account concurrency → other functions throttle. **Fix:** Set reserved concurrency per function, especially for critical paths.

7. **Lambda@Edge cold starts:** Edge functions run in CloudFront POPs, not your primary region. IAM auth, secret fetching all have to be baked in or fetched at cold start from the edge — latency implications.

---

## 1.8 Performance Deep Dive

| Factor | Detail |
|--------|--------|
| Cold start (Node.js) | 50–400ms without VPC; 300ms–1s with VPC (hyperplane) |
| Cold start (Java) | 1–8 seconds (JVM startup + class loading) |
| Warm invocation | ~1ms overhead from Lambda runtime |
| Max throughput | Concurrency × RPS per instance (e.g., 1,000 concurrent × 100 RPS = 100k RPS) |
| Memory/CPU | 1,769MB = 1 vCPU; 3,008MB = ~1.7 vCPU; linear between |
| Ephemeral storage | 512MB–10GB (/tmp); billed per GB-second |
| Init code (module level) | Runs every cold start; amortized across warm invocations |
| Network latency | ~0.2ms internal AWS; same VPC = microseconds |

**Latency Sources:**
1. TLS handshake to Lambda endpoint (~10ms for new connection)
2. SigV4 signature verification (~1ms)
3. Worker Manager scheduling (~1ms warm, ~100ms cold)
4. MicroVM init if cold start (~125ms Firecracker boot)
5. Runtime init (Node.js module loading, your init code)
6. Handler execution
7. Log flushing (async, not on critical path)

---

## 1.9 Cost Deep Dive

**Billing formula:**
```
Cost = (Request count × $0.20/million)
     + (GB-seconds × $0.0000166667/GB-second)

GB-seconds = (RAM_MB / 1024) × duration_seconds × invocations
```

**Hidden costs engineers miss:**
1. **Provisioned concurrency:** Pre-warmed sandboxes billed per hour even with zero requests (~$0.015/GB-hour). Often more expensive than EC2 for constant traffic.
2. **Data transfer:** Lambda to internet = $0.09/GB. Lambda to DynamoDB/S3 in same region = free. Lambda in VPC to NAT Gateway = $0.045/GB + $0.01/GB NAT processing fee.
3. **ENI hours for VPC Lambda:** Each Lambda in VPC uses a hyperplane ENI that is billed as part of VPC pricing.
4. **CloudWatch Logs ingestion:** $0.50/GB. High-frequency logging from Lambda = major bill driver.
5. **X-Ray traces:** If enabled, $5/million traces after free tier.

---

## 1.10 Comparison

| | Lambda | ECS Fargate | EC2 |
|-|--------|-------------|-----|
| Startup | 100ms–2s cold | 30–90s (container) | 60–300s |
| Max duration | 15 min | Unlimited | Unlimited |
| State | Ephemeral | Ephemeral (per task) | Persistent |
| Scaling | Automatic, per-request | Task-level, minutes | ASG, minutes |
| Cost model | Per-ms | Per-second (task) | Per-hour |
| Best for | Event-driven, bursty | Microservices, APIs | Long-running, stateful |

---

# 2. API GATEWAY

## 2.1 Core Purpose

**Problem solved:** Managed HTTP ingress layer that handles TLS termination, authentication, throttling, request routing, and transformation — so your backends don't need to implement these cross-cutting concerns.

**What breaks without it:** You'd need nginx/Envoy/HAProxy on EC2 with custom auth middleware, rate limiting, CORS handling, API versioning, and SDK generation — all undifferentiated heavy lifting.

---

## 2.2 Internal Architecture

### Three Distinct Products (Often Confused)

```
┌─────────────────────────────────────────────────────────────┐
│                    API GATEWAY FAMILY                        │
├──────────────────┬──────────────────┬───────────────────────┤
│  REST API (v1)   │  HTTP API (v2)   │  WebSocket API        │
│                  │                  │                        │
│  Full features   │  Lower latency   │  Persistent           │
│  Request/resp    │  Cheaper (~70%)  │  connections          │
│  transform       │  No transform    │  Route by action      │
│  WAF integration │  JWT auth native │                        │
│  API Keys/usage  │  VPC Link v2     │                        │
│  plans           │                  │                        │
└──────────────────┴──────────────────┴───────────────────────┘
```

### Control Plane vs Data Plane

```
CONTROL PLANE                           DATA PLANE
─────────────────────                   ─────────────────────────────
- CreateRestApi, CreateResource         - Regional endpoint handlers
- Deploy (creates immutable Deployment) - Edge-optimized (CloudFront)
- Stage configuration                   - Private (VPC Endpoint)
- Authorizer configuration              - Request pipeline processor
- Usage plan management                 - Lambda integration runner
- Model/schema registry                 - HTTP integration proxy
```

### Request Lifecycle (REST API, detailed)

```
1. Client → api-id.execute-api.region.amazonaws.com/stage/resource
2. Edge-Optimized: CloudFront POP terminates TLS → routes to regional API GW
   Regional: Direct to regional API GW fleet
3. API GW frontend fleet:
   a. TLS termination (not re-encrypted inside unless private integration)
   b. Host-based routing → identifies API ID + Stage
   c. Rate limiting check: token bucket per account/stage/method
      - Account level: 10,000 RPS default
      - Stage level: configurable
      - Method level: configurable
   d. Authorizer evaluation:
      - IAM: SigV4 signature → IAM policy evaluation → allow/deny
      - Lambda authorizer: invoke authorizer Lambda → cache result by token (TTL configurable)
      - Cognito JWT: validate JWT signature against JWKS endpoint (cached)
4. Request mapping: VTL (Velocity Template Language) transforms input
   - $input.json('$.field') → maps request body fields
   - Headers/query params injected into integration request
5. Integration invocation:
   - Lambda Proxy: async Invoke API call to Lambda
   - Lambda Non-Proxy: VTL-transformed payload to Lambda, VTL-transformed response back
   - HTTP/HTTP_PROXY: TCP to backend (not Lambda)
   - AWS Service: direct service API call (e.g., SQS SendMessage)
   - Mock: response generated without backend call
6. Response mapping (non-proxy): VTL transforms integration response
7. Response returned to client
   - CORS headers injected if configured
   - Stage variables resolved
```

### Internal Multi-Tenancy

API Gateway frontend is a **shared fleet** behind CloudFront (for edge-optimized). Your API is identified by a unique API ID embedded in the hostname. The fleet:
- Uses consistent hashing on API ID to route to the right configuration shard
- Configuration is replicated to all regional fleet nodes (control plane writes → data plane reads)
- Your requests share compute with other customers' requests; rate limits are the isolation mechanism

### Lambda Authorizer Internals

```
Request with token
       │
       ▼
API GW checks authorizer cache (key = token, TTL = 0-3600s)
       │
     HIT ──→ use cached IAM policy
       │
    MISS ──→ Invoke authorizer Lambda synchronously
              │
              ▼
         Authorizer returns:
         {
           "principalId": "user-123",
           "policyDocument": { ... IAM policy ... },
           "context": { "userId": "123" }  // passed to backend
         }
              │
              ▼
         API GW caches policy (per token)
         API GW evaluates policy against method ARN
         API GW injects context into request headers ($context.authorizer.userId)
```

---

## 2.3 Invocation & Communication Model

- **Push model:** Clients push HTTP requests to API GW
- **Synchronous:** Client waits for response (29-second hard timeout on REST API)
- **Request-driven:** Not event-driven; API GW is a synchronous passthrough
- **Backpressure:** Token bucket throttling; returns 429 TooManyRequests when bucket empty

---

## 2.4 Deep-Dive Scenario (Node.js)

```javascript
// Lambda integration with context injection
exports.handler = async (event) => {
  // API GW Proxy integration gives you raw event:
  console.log(event.httpMethod);          // GET, POST, etc.
  console.log(event.pathParameters);      // {userId: "123"}
  console.log(event.queryStringParameters); // {page: "1"}
  console.log(event.headers);            // all HTTP headers
  console.log(event.body);               // raw body string (must JSON.parse)
  console.log(event.requestContext.authorizer.userId); // from Lambda authorizer
  console.log(event.requestContext.requestId); // for tracing
  
  // CRITICAL: body is a STRING, not parsed object
  const body = event.isBase64Encoded 
    ? Buffer.from(event.body, 'base64').toString() 
    : event.body;
  
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',    // CORS
    },
    body: JSON.stringify({ result: 'ok' }),
    // isBase64Encoded: false  // for binary responses
  };
};
```

---

## 2.5 When to Use

| Use When | Tradeoff |
|----------|---------|
| Serverless API (Lambda backend) | 29-second hard timeout |
| Need API keys + usage plans | REST API only; HTTP API lacks usage plans |
| Native AWS auth (SigV4/Cognito) | VTL transformation has steep learning curve |
| WebSocket APIs | Connection state stored externally; no server-side push without @connections |
| Direct AWS service integration | Can call SQS/DynamoDB directly without Lambda intermediary |

## 2.6 When NOT to Use

- **Long polling / streaming responses:** 29-second timeout kills SSE (Server-Sent Events) or streaming JSON
- **High-volume low-latency APIs:** ALB + ECS is cheaper and faster at scale (API GW adds ~5-10ms per request)
- **Binary protocol APIs (gRPC):** API GW doesn't support HTTP/2 end-to-end; use ALB with gRPC target groups
- **Very high throughput >100k RPS sustained:** ALB scales more cost-effectively

## 2.7 Common Pitfalls

1. **29-second timeout gotcha:** Lambda can run up to 15 minutes, but API GW will 504 at 29 seconds. Client sees error; Lambda keeps running — wasted compute.
2. **Lambda authorizer caching trap:** Token cached → permissions change → old policy used for TTL duration. Set TTL=0 for security-sensitive operations.
3. **Binary media types:** Without configuring `binaryMediaTypes` in the API, binary content gets double-base64 encoded.
4. **Payload size limits:** 10MB max request/response for REST; 6MB async payload via Lambda.
5. **Missing stage variables in dev/prod:** Using hardcoded URLs instead of `${stageVariables.backendUrl}` → requires code change per environment.

## 2.8 Performance Deep Dive

| Factor | REST API | HTTP API |
|--------|---------|---------|
| Added latency | ~10-15ms | ~5-8ms |
| Max payload | 10MB | 10MB |
| Timeout | 29s hard | 29s |
| Throttling | Token bucket | Token bucket |
| Caching | CloudFront (edge-optimized) or ElastiCache | No built-in caching |

**Latency sources:**
1. TLS handshake (first request: ~50ms; keepalive: ~1ms)
2. Host routing + API ID lookup (~1ms)
3. Rate limit check (~0.5ms)
4. Authorizer invocation (Lambda: +50-100ms; cached: ~0ms)
5. VTL transformation (~1-5ms per mapping)
6. Integration invocation (Lambda: +3-5ms SDK overhead)

## 2.9 Cost Deep Dive

```
REST API:  $3.50/million requests + $0.09/GB data transfer
HTTP API:  $1.00/million requests + $0.09/GB data transfer
WebSocket: $1.00/million messages + $0.25/million connection-minutes

Hidden costs:
- CloudWatch Logs: $0.50/GB for access logs
- Cache: $0.02–$3.84/hour depending on cache size
- Lambda authorizer invocations: billed per invoke
- Data transfer out to internet: $0.09/GB
```

## 2.10 Comparison

| | API GW REST | API GW HTTP | ALB |
|-|------------|-------------|-----|
| Latency overhead | ~10-15ms | ~5-8ms | ~1-2ms |
| Cost | High | Moderate | Low |
| Features | Full (WAF, caching, transforms) | Minimal | Moderate |
| WebSocket | Yes | No | No |
| gRPC | No | No | Yes |
| Backend types | Lambda, HTTP, AWS | Lambda, HTTP | EC2, ECS, Lambda, IPs |

---

# 3. AMAZON S3

## 3.1 Core Purpose

**Problem solved:** Infinitely scalable, durable object storage with a flat key-value namespace — eliminating the need to manage distributed file systems, replication, and capacity planning.

**What breaks:** You'd need Ceph/GlusterFS clusters, manage disk failures, build replication across AZs, implement object versioning and lifecycle policies yourself.

---

## 3.2 Internal Architecture

### S3 Internal Structure

S3 is NOT a traditional filesystem. Internally:

```
S3 Internals:
─────────────────────────────────────────────────────────────────

CONTROL PLANE (Metadata layer)
┌──────────────────────────────────────────────────────┐
│ Bucket index shards (key → location mapping)          │
│  - Billions of objects distributed across shards      │
│  - Each shard handles a subset of key-space          │
│  - Shard = internal metadata server (similar to      │
│    a B-tree node managing key ranges)                │
└──────────────────────────────────────────────────────┘

DATA PLANE (Storage layer)
┌──────────────────────────────────────────────────────┐
│ Storage nodes (physical HDD/SSD servers)              │
│  - Objects split into "chunks" (internally)          │
│  - Each chunk replicated 3+ times across AZs         │
│  - Chunks distributed via consistent hashing         │
│  - S3 uses erasure coding for large objects          │
└──────────────────────────────────────────────────────┘
```

### PUT Object Flow (Detailed)

```
1. Client → PUT /bucket/key HTTP request to s3.amazonaws.com
2. S3 front-end fleet: TLS termination, SigV4 auth, bucket policy eval
3. Front-end identifies bucket → locates metadata shard for this key range
4. S3 hashes the key to determine data placement:
   - hash(bucket + key) → identifies primary storage nodes
   - Data simultaneously streamed to 3+ storage nodes (synchronous write to majority)
5. Object chunks written to storage nodes' local disk
6. When majority (2 of 3) acknowledge write: HTTP 200 returned to client
7. Third replica written asynchronously (not on critical path)
8. Metadata shard updated: key → {locations, ETag, size, version_id, storage_class}
9. If versioning enabled: previous version ID preserved in metadata
10. If event notifications configured: EventBridge/SNS/SQS receives s3:ObjectCreated event
```

### Key Hashing and Prefix Performance

This is where most engineers get it wrong:

```
HISTORICAL (pre-2018): S3 stored keys in lexicographic sorted order.
  - Keys starting with "2024/01/" would all hit same shard
  - Solution: random prefix (MD5 hash prefix)

CURRENT (post-2018): S3 automatically partitions based on request rate.
  - No longer need random prefixes for performance
  - S3 can automatically split hot prefixes into multiple shards
  - 5,500 GET/HEAD + 3,500 PUT/POST/DELETE per second PER PREFIX
  - Prefix = the key up to any delimiter (e.g., "images/2024/01/")
```

### Multi-AZ Durability

```
AZ-A ──────┐
            ├── Data written synchronously to 2 of 3 AZs before 200 OK
AZ-B ──────┤    Third AZ receives async copy
            │
AZ-C ──────┘    (S3 One Zone-IA: single AZ only, 99.5% availability)

11 nines durability = 10^-11 annual loss probability
= you can store 10 million objects for 10,000 years and expect to lose ONE
```

### Multipart Upload Internals

```
Large object (>100MB recommended, required >5GB):
1. InitiateMultipartUpload → returns UploadId
2. Client uploads N parts (min 5MB each, max 10,000 parts)
   - Parts uploaded in parallel to different S3 front-ends
   - Each part acknowledged independently
3. CompleteMultipartUpload → S3 stitches parts into final object
   - Internal metadata records part locations
   - Final object assembled without data copy (metadata-level join)
4. If not completed: parts consume storage but object doesn't exist
   - Lifecycle policy: abort incomplete multipart uploads after N days
```

---

## 3.3 Invocation & Communication Model

- **Request-driven:** Clients push GET/PUT requests
- **Event-driven outbound:** S3 emits events on object operations
- **Strong consistency (post-December 2020):** PUT then immediate GET returns the new object (previously eventually consistent)
- **Conditional requests:** `If-None-Match`, `If-Modified-Since` for cache validation

---

## 3.4 Deep-Dive Scenario (Node.js)

```javascript
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const { Upload } = require('@aws-sdk/lib-storage');

const s3 = new S3Client({ region: 'us-east-1' });

// Large file upload with multipart
const uploadLargeFile = async (fileStream, bucket, key) => {
  const upload = new Upload({
    client: s3,
    params: {
      Bucket: bucket,
      Key: key,
      Body: fileStream,
      ServerSideEncryption: 'aws:kms',
      SSEKMSKeyId: process.env.KMS_KEY_ID,
    },
    queueSize: 4,      // 4 concurrent part uploads
    partSize: 10 * 1024 * 1024,  // 10MB parts
    leavePartsOnError: false,    // abort if failed
  });

  upload.on('httpUploadProgress', (progress) => {
    console.log(`Uploaded: ${progress.loaded}/${progress.total}`);
  });

  return upload.done();
};

// Presigned URL for client-side upload (bypasses Lambda 6MB limit)
const getPresignedUploadUrl = async (bucket, key, expiresIn = 300) => {
  const command = new PutObjectCommand({ Bucket: bucket, Key: key });
  return getSignedUrl(s3, command, { expiresIn });
  // URL includes SigV4 signature in query params
  // Client uploads directly to S3 — Lambda never touches the bytes
};

// Streaming download
const streamFromS3 = async (bucket, key) => {
  const response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
  // response.Body is a ReadableStream (Node.js)
  return response.Body;
};
```

---

## 3.5 When to Use

| Use When | Tradeoff |
|----------|---------|
| Static assets (JS/CSS/images) | No server-side computation; pair with CloudFront |
| Backup/archival | Lifecycle to Glacier; retrieval latency |
| Data lake (Parquet/ORC files) | Eventual namespace; use Athena/Glue for querying |
| Large file storage (video, ML datasets) | Egress costs at scale |
| Event-driven pipelines (upload triggers Lambda) | S3 event notification delivery is at-least-once |

## 3.6 When NOT to Use

- **Low-latency random access patterns:** S3 latency is 20-100ms first byte (not for sub-ms IOPS)
- **Frequent small writes:** Each PUT = full HTTP roundtrip; use EFS or EBS for frequent appends
- **Transactional storage:** No ACID; concurrent writes to same key = last-writer-wins
- **Structured data with queries:** Use RDS or DynamoDB; S3 Select is limited

## 3.7 Common Pitfalls

1. **Public buckets:** ACL "public-read" + no bucket policy = world-readable. Use S3 Block Public Access at account level.
2. **S3 event notification ordering:** Events are at-least-once. Two rapid PUTs may only trigger one notification or deliver them out of order.
3. **Cost explosion from list operations:** `ListObjectsV2` on bucket with millions of objects + paginating = thousands of API calls at $0.005/1k.
4. **Not using Transfer Acceleration:** Cross-region uploads go over public internet; Transfer Acceleration routes through CloudFront edge → AWS backbone.
5. **Versioning without lifecycle:** Every PUT adds a new version; costs grow unboundedly. Add lifecycle rule to expire non-current versions.
6. **Missing multipart abort policy:** Failed multipart uploads leave orphaned parts. Add lifecycle rule: `AbortIncompleteMultipartUpload` after 7 days.

## 3.8 Performance Deep Dive

| Metric | Value |
|--------|-------|
| First byte latency | 20–100ms (single-digit ms with S3 Express One Zone) |
| Throughput per prefix | 5,500 GET + 3,500 PUT per second |
| Max object size | 5TB |
| Part size (multipart) | 5MB–5GB per part, max 10,000 parts |
| S3 Express One Zone | Single-digit ms, same-AZ only, 10x cost |

## 3.9 Cost Deep Dive

```
Standard storage:     $0.023/GB-month
Glacier Instant:      $0.004/GB-month  (retrieval: ms but $0.03/GB)
Glacier Flexible:     $0.0036/GB-month (retrieval: 3-5 hours)
Deep Archive:         $0.00099/GB-month (retrieval: 12+ hours)

Request costs (often overlooked):
  PUT/COPY/POST: $0.005/1,000 requests
  GET/SELECT:    $0.0004/1,000 requests
  LIST:          $0.005/1,000 requests (same as PUT!)

Data transfer:
  Inbound: FREE
  To internet: $0.09/GB first 10TB, decreasing tiers
  To CloudFront (same region): FREE
  To EC2/Lambda (same region): FREE
  Cross-region: $0.02/GB

Replication:
  Cross-region replication: $0.015/GB replicated + destination storage
```

---

# 4. AMAZON DYNAMODB

## 4.1 Core Purpose

**Problem solved:** A fully managed, horizontally scalable NoSQL key-value and document database that provides consistent single-digit millisecond latency at any scale, eliminating the need to manage database servers, replication, or sharding.

**What breaks:** You'd shard MySQL/PostgreSQL manually (Vitess/PlanetScale), manage replication lag, handle failover, and still struggle to achieve <10ms at millions of RPS.

---

## 4.2 Internal Architecture

### The Dynamo Paper Lineage

DynamoDB's lineage traces to the 2007 Dynamo paper (eventually consistent key-value store). DynamoDB improved on it with:
- Strong consistency option (quorum reads)
- ACID transactions (2PC across partitions)
- Global tables (multi-region active-active)
- DAX (write-through cache layer)

### Physical Architecture

```
DynamoDB Table
       │
       ├── Partition 1  (key range: hash 0x000...–0x3FF...)
       │       ├── Storage Node A (primary)    ← writes go here first
       │       ├── Storage Node B (replica)    ← async replication
       │       └── Storage Node C (replica)    ← async replication
       │
       ├── Partition 2  (key range: hash 0x400...–0x7FF...)
       │       ├── Storage Node D (primary)
       │       ├── Storage Node E (replica)
       │       └── Storage Node F (replica)
       │
       └── ...N partitions based on table size and throughput
```

### Partitioning Logic (The Real Algorithm)

```
1. Partition Key is hashed using MD5 (128-bit output)
   hash_value = MD5(partition_key)
   
2. Hash value mapped to partition ring (consistent hashing)
   - Ring divided into N virtual nodes (vnodes)
   - Each physical partition node owns multiple vnodes
   
3. Key routing:
   physical_partition = hash_value / (MAX_HASH / num_partitions)
   
4. Number of partitions determined by:
   max(ceil(table_size_GB / 10), ceil(provisioned_WCU / 1000), ceil(provisioned_RCU / 3000))
   
5. Each partition:
   - Max 10GB data
   - Max 3,000 RCU (read capacity units)
   - Max 1,000 WCU (write capacity units)
   
6. Partition SPLIT: when any limit exceeded, DynamoDB splits the partition
   - New partition inherits half the key range
   - Data migrated in background
   - Brief throttling possible during split
   
7. Partition MERGE: DynamoDB merges underutilized partitions
   - After on-demand table peaks, partitions may be merged
```

### Request Lifecycle

```
1. Client SDK → DynamoDB regional endpoint (HTTPS)
2. Request Router: TLS termination, SigV4 auth, table existence check
3. Metadata Service lookup: table_id → partition map (cached on request router)
4. Consistent hashing: hash(partition_key) → partition_id → storage node address
5. Request forwarded to primary storage node
6. Primary writes to B-tree log (WAL: Write-Ahead Log)
7. Replication to 2 other storage nodes (Paxos-based consensus for strong consistency)
8. When quorum (2/3) acknowledges: write confirmed
9. Response returned to client
10. Async: DynamoDB Streams event written (if enabled)
```

### Storage Node Internals

Each storage node runs:
- **B-tree engine:** Keys sorted by sort key within each partition key
- **WAL (Write-Ahead Log):** All writes go to WAL first (durability)
- **Memtable:** Recent writes in memory for fast reads
- **SSTables:** Compacted on-disk storage (LSM-tree style)

### Read Consistency Internals

```
Eventually Consistent Read (default):
  - Request goes to any of 3 storage nodes (round-robin)
  - Cheaper: 0.5 RCU per 4KB
  - May read stale data (replication lag: typically <1 second)

Strongly Consistent Read:
  - Request goes to PRIMARY storage node only
  - Quorum read: waits for most recent committed value
  - Costs 1 RCU per 4KB
  - Higher latency if primary is temporarily overloaded

Transactional Read:
  - DynamoDB uses 2-Phase Locking across partitions
  - Coordinator node manages transaction
  - Costs 2 RCU per 4KB
```

---

## 4.3 Invocation & Communication Model

- **Request-driven:** SDK makes HTTP/HTTPS calls to DynamoDB endpoint
- **Synchronous:** All operations return immediately or timeout
- **Streams:** Asynchronous event emission; Lambda ESM polls streams
- **Backpressure:** Throttling (ProvisionedThroughputExceededException) when WCU/RCU exhausted

---

## 4.4 Deep-Dive Scenario (Node.js)

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand, QueryCommand, 
        TransactWriteCommand, UpdateCommand } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({ region: 'us-east-1' });
const ddb = DynamoDBDocumentClient.from(client, {
  marshallOptions: { removeUndefinedValues: true }
});

// ===== SINGLE TABLE DESIGN =====
// Table: "App"
// PK: entityType#id (e.g., "USER#user-123")
// SK: metadata#timestamp (e.g., "ORDER#2024-01-15T10:00:00Z")
// GSI1: inverted index for reverse lookups

// Write an order (idempotent with condition)
const createOrder = async (order) => {
  await ddb.send(new PutCommand({
    TableName: 'App',
    Item: {
      PK: `USER#${order.userId}`,
      SK: `ORDER#${order.orderId}`,
      GSI1PK: `STATUS#${order.status}`,
      GSI1SK: `DATE#${order.createdAt}`,
      ...order,
      TTL: Math.floor(Date.now() / 1000) + 90 * 24 * 3600  // expire in 90 days
    },
    ConditionExpression: 'attribute_not_exists(PK) AND attribute_not_exists(SK)'
  }));
};

// Atomic counter increment (no read-modify-write race condition)
const incrementView = async (articleId) => {
  const result = await ddb.send(new UpdateCommand({
    TableName: 'App',
    Key: { PK: `ARTICLE#${articleId}`, SK: 'METADATA' },
    UpdateExpression: 'SET viewCount = if_not_exists(viewCount, :zero) + :inc',
    ExpressionAttributeValues: { ':zero': 0, ':inc': 1 },
    ReturnValues: 'UPDATED_NEW'
  }));
  return result.Attributes.viewCount;
};

// ACID transaction across two entities
const transferBalance = async (fromId, toId, amount) => {
  await ddb.send(new TransactWriteCommand({
    TransactItems: [
      {
        Update: {
          TableName: 'App',
          Key: { PK: `ACCOUNT#${fromId}`, SK: 'BALANCE' },
          UpdateExpression: 'SET balance = balance - :amount',
          ConditionExpression: 'balance >= :amount',
          ExpressionAttributeValues: { ':amount': amount }
        }
      },
      {
        Update: {
          TableName: 'App',
          Key: { PK: `ACCOUNT#${toId}`, SK: 'BALANCE' },
          UpdateExpression: 'SET balance = balance + :amount',
          ExpressionAttributeValues: { ':amount': amount }
        }
      }
    ]
  }));
};

// Query with pagination (critical for large datasets)
const getUserOrders = async (userId, lastEvaluatedKey = null) => {
  const params = {
    TableName: 'App',
    KeyConditionExpression: 'PK = :pk AND begins_with(SK, :prefix)',
    ExpressionAttributeValues: {
      ':pk': `USER#${userId}`,
      ':prefix': 'ORDER#'
    },
    Limit: 20,  // page size
    ScanIndexForward: false,  // newest first
  };
  if (lastEvaluatedKey) params.ExclusiveStartKey = lastEvaluatedKey;
  
  return ddb.send(new QueryCommand(params));
  // result.LastEvaluatedKey → pass to next call for pagination
  // CRITICAL: Limit applies BEFORE FilterExpression filtering
};
```

---

## 4.5 When to Use

| Use When | Tradeoff |
|----------|---------|
| Known access patterns (pre-modeled) | Schema change = rewrite table; no ALTER TABLE |
| Single-digit ms at any scale | No ad-hoc complex queries (no JOIN, limited aggregation) |
| Auto-scaling write throughput needed | On-demand mode costs 6x per RCU/WCU vs provisioned |
| TTL-based expiry (sessions, cache) | TTL deletion is async (actual deletion up to 48h after expiry) |
| Serverless architectures | Minimum cost floor per table |
| Global multi-region active-active | Replication async; last-writer-wins per item |

## 4.6 When NOT to Use

- **Complex relational queries:** JOINs across entities = multiple DynamoDB calls + application-side join = slow
- **Reporting/analytics:** Full-table scans expensive (use S3 + Athena for analytics)
- **Unknown access patterns:** You'll redesign the table 3 times as requirements evolve
- **Small data + complex queries:** Aurora Serverless v2 is cheaper and more flexible at small scale

## 4.7 Common Pitfalls

1. **Hot partition:** All writes hitting same partition key (e.g., partition key = "status" with value "PENDING"). Max 1,000 WCU/partition. Solution: high-cardinality partition key or write sharding.

2. **Scan on large tables:** Full table scan = reads every item = every partition = massive cost. Production tables should NEVER use Scan except for migration.

3. **Limit applies before FilterExpression:** `Limit: 10` with a filter might return 0 results after filtering (10 items read, 0 match filter). Requires multiple pages to find results.

4. **GSI eventual consistency:** Query on GSI returns data within ~1 second of write. Don't query GSI immediately after write for critical paths.

5. **Transaction conflicts at high concurrency:** TransactWriteItems uses optimistic concurrency. High-contention writes (same item from many Lambda instances) → frequent `TransactionCanceledException`. Use conditional updates instead.

6. **On-demand mode cost spike:** Burst to 2x previous peak handled free. Beyond that: throttling. On-demand charges $1.25/million WCU (vs $0.65 provisioned with auto-scaling) — 2x cost at sustained high load.

## 4.8 Performance Deep Dive

| Metric | Value |
|--------|-------|
| Read latency | <5ms (P99 single-digit ms) |
| Write latency | <5ms |
| Max item size | 400KB |
| Max partition throughput | 3,000 RCU or 1,000 WCU |
| Burst capacity | Last 5 minutes of unused capacity (up to 300 seconds) |
| GSI write amplification | Every write to table = write to all GSIs |
| Transaction overhead | 2x the WCU/RCU cost + coordination latency |

## 4.9 Cost Deep Dive

```
On-demand:
  Write: $1.25/million write request units
  Read:  $0.25/million read request units

Provisioned (with auto-scaling):
  Write: $0.00065/WCU-hour = $0.47/WCU-month
  Read:  $0.00013/RCU-hour = $0.09/RCU-month
  
Storage: $0.25/GB-month
DynamoDB Streams: $0.02/100,000 stream reads
Global Tables replication: $0.109/million replicated WCU
DAX: Node-hour pricing ($0.269/hour for cache.r4.large)

Hidden cost: GSI storage and throughput
  - Each GSI has its own WCU/RCU allocation
  - 5 GSIs with 100 WCU each = 500 extra WCU
  - GSI write amplification: 1 table write = 1 write per GSI
```

---

# 5. AMAZON SQS

## 5.1 Core Purpose

**Problem solved:** Durable, distributed message queue that decouples producers from consumers, providing backpressure, retry, and at-least-once delivery guarantees — eliminating the need to build custom message buffering.

**What breaks:** Producer overwhelms consumer → consumer crashes → data lost. Without SQS: custom Redis-based queuing with manual ACK/retry logic, no DLQ, no visibility timeout.

---

## 5.2 Internal Architecture

### Queue Types

```
Standard Queue:
  - Nearly unlimited throughput (millions/second)
  - At-least-once delivery (may deliver same message 2x)
  - Best-effort ordering (NOT FIFO)
  - Messages distributed across redundant servers

FIFO Queue:
  - 300 TPS (3,000 with batching)
  - Exactly-once processing (deduplication)
  - Strict per-MessageGroupId ordering
  - Higher internal coordination overhead
```

### Internal Storage Architecture

```
SQS Message Flow:
┌──────────────┐       ┌─────────────────────────────────────┐
│   Producer   │──PUT──▶  SQS Frontend (regional fleet)       │
└──────────────┘       │                                     │
                       │  ┌─────────────────────────────┐    │
                       │  │  Message Store               │    │
                       │  │  (redundant across 3 AZs)    │    │
                       │  │                             │    │
                       │  │  - Messages stored as blobs │    │
                       │  │  - 256KB max per message    │    │
                       │  │  - Retention: 1 min–14 days │    │
                       │  └─────────────────────────────┘    │
                       └─────────────────────────────────────┘
                                       │
                       ┌───────────────▼────────────────────┐
                       │  Consumer POLL (ReceiveMessage)     │
                       │  - Returns 1–10 messages            │
                       │  - Sets visibility timeout          │
                       │  - Message "in-flight" (invisible)  │
                       └────────────────────────────────────┘
```

### Visibility Timeout Mechanics

```
Timeline:
T=0:   Producer sends message M
T=1:   Consumer A receives M (visibility timeout = 30s)
       M is now INVISIBLE to other consumers
T=15:  Consumer A still processing
T=30:  VISIBILITY TIMEOUT EXPIRES
       M becomes VISIBLE again
T=31:  Consumer B receives M (duplicate delivery!)
       
CRITICAL: If your processing takes > visibility timeout:
  - Message is redelivered
  - If consumer A completes at T=32 and calls DeleteMessage:
    DeleteMessage succeeds (M deleted using receipt handle)
  - Consumer B's processing = wasted work (but no data loss)
  
Fix: ChangeMessageVisibility before timeout expires
     Or set visibility timeout to max expected processing time × 2
```

### Dead Letter Queue (DLQ) Mechanics

```
Message M processed N times (maxReceiveCount):
  Each RECEIVE increments ApproximateReceiveCount attribute
  When ApproximateReceiveCount >= maxReceiveCount:
    SQS moves message to DLQ (not a copy — moved)
    DLQ must be same type (Standard→Standard, FIFO→FIFO)
    DLQ message retention separate from source queue

DLQ Redrive (re-process DLQ messages):
  Move messages from DLQ back to source queue
  - Preserves original message body and attributes
  - Resets receive count to 0
```

### Long Polling vs Short Polling

```
Short polling (WaitTimeSeconds=0):
  - Returns immediately even if no messages
  - Samples random subset of SQS servers
  - May return empty even if messages exist
  - Higher cost ($0.40/million requests)

Long polling (WaitTimeSeconds=1-20):
  - Connection held open up to N seconds
  - Returns as soon as message arrives
  - Queries ALL SQS servers before returning empty
  - Reduces empty receives = lower cost
  - RECOMMENDED for all consumers
```

---

## 5.3 Invocation & Communication Model

- **Pull model:** Consumers poll SQS (SQS does NOT push to consumers)
- **Async:** Producer fire-and-forget; consumer polls independently
- **At-least-once (Standard) / Exactly-once (FIFO):**
- **Backpressure:** Consumer controls rate by controlling poll frequency; unprocessed messages accumulate in queue (natural backpressure signal)

---

## 5.4 Deep-Dive Scenario (Node.js)

```javascript
const { SQSClient, ReceiveMessageCommand, DeleteMessageCommand,
        ChangeMessageVisibilityCommand, SendMessageBatchCommand } = require('@aws-sdk/client-sqs');

const sqs = new SQSClient({ region: 'us-east-1' });
const QUEUE_URL = process.env.QUEUE_URL;
const VISIBILITY_TIMEOUT = 60; // seconds — set > max processing time

// Long-polling consumer loop (for ECS/EC2 — not Lambda)
const pollAndProcess = async () => {
  while (true) {
    const response = await sqs.send(new ReceiveMessageCommand({
      QueueUrl: QUEUE_URL,
      MaxNumberOfMessages: 10,       // batch of 10
      WaitTimeSeconds: 20,           // long poll
      VisibilityTimeout: VISIBILITY_TIMEOUT,
      AttributeNames: ['ApproximateReceiveCount', 'SentTimestamp'],
      MessageAttributeNames: ['All'],
    }));

    if (!response.Messages?.length) continue;

    await Promise.allSettled(
      response.Messages.map(async (message) => {
        const body = JSON.parse(message.Body);
        
        // Extend visibility before timeout if processing is slow
        const extendTimer = setInterval(async () => {
          await sqs.send(new ChangeMessageVisibilityCommand({
            QueueUrl: QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle,
            VisibilityTimeout: VISIBILITY_TIMEOUT,
          }));
        }, (VISIBILITY_TIMEOUT - 5) * 1000);

        try {
          await processMessage(body);
          
          // DELETE only after successful processing
          await sqs.send(new DeleteMessageCommand({
            QueueUrl: QUEUE_URL,
            ReceiptHandle: message.ReceiptHandle,
          }));
        } catch (err) {
          console.error('Processing failed:', err);
          // DO NOT delete → message reappears after visibility timeout
          // After maxReceiveCount failures → goes to DLQ
        } finally {
          clearInterval(extendTimer);
        }
      })
    );
  }
};

// Batch producer for throughput (up to 10 messages per API call)
const enqueueBatch = async (items) => {
  const chunks = [];
  for (let i = 0; i < items.length; i += 10) {
    chunks.push(items.slice(i, i + 10));
  }
  
  for (const chunk of chunks) {
    const result = await sqs.send(new SendMessageBatchCommand({
      QueueUrl: QUEUE_URL,
      Entries: chunk.map((item, idx) => ({
        Id: `msg-${idx}`,
        MessageBody: JSON.stringify(item),
        MessageAttributes: {
          eventType: { DataType: 'String', StringValue: item.type }
        }
      }))
    }));
    
    if (result.Failed?.length) {
      console.error('Batch failures:', result.Failed);
      // Retry failed messages individually
    }
  }
};
```

---

## 5.5 When to Use

| Use When | Tradeoff |
|----------|---------|
| Decoupling async services | No ordering guarantee (Standard queue) |
| Absorbing traffic spikes | Queue depth grows; need CloudWatch alarm + auto-scale |
| Dead letter handling | DLQ requires separate monitoring and redrive strategy |
| At-least-once processing | Must make consumers idempotent |
| Fan-out via SNS → SQS | Multiple SQS queues per SNS topic |

## 5.6 When NOT to Use

- **Ordered message processing:** Use SQS FIFO (300 TPS limit) or Kinesis (sharded, ordered)
- **Real-time streaming:** Kinesis for high-throughput ordered streams with replay
- **Message size >256KB:** Use S3 + SQS pointer pattern (store payload in S3, send S3 key in SQS)
- **Request-reply patterns:** SQS is one-way; for RPC use correlation IDs + reply queues (complex)
- **Long-term message retention >14 days:** SQS max retention is 14 days; use S3 or DynamoDB for durable state

## 5.7 Common Pitfalls

1. **Poison pill messages:** Message fails processing every time → hits DLQ after maxReceiveCount. Without DLQ: message blocks behind visible, gets redelivered forever, burning consumer CPU.
2. **Not making consumers idempotent:** At-least-once delivery means same message delivered 2x. Duplicate payment processing = disaster.
3. **Visibility timeout too short:** Processing takes 45s, timeout = 30s → duplicate delivery while first consumer is still processing.
4. **Large messages without S3 extended client:** 256KB limit. Solution: SQS Extended Client stores payload in S3, sends S3 reference in SQS message.
5. **Standard queue in FIFO context:** Using Standard queue expecting order → messages processed out of order → inventory goes negative.

## 5.8 Performance Deep Dive

| Metric | Standard | FIFO |
|--------|---------|------|
| Throughput | Unlimited | 300 TPS (3,000 batched) |
| Message size | 256KB | 256KB |
| Retention | 1min–14 days | 1min–14 days |
| In-flight messages | 120,000 | 20,000 |
| Long poll max wait | 20s | 20s |
| Latency | <10ms | <10ms |

## 5.9 Cost

```
Standard: $0.40/million requests
FIFO:     $0.50/million requests

Note: Each ReceiveMessage API call = 1 request (returns up to 10 messages)
Empty receive (long poll returned nothing) = still 1 request

Hidden cost: SQS Extended Client S3 operations
  - Every large message = S3 PUT + S3 GET (both billed)
```

---

# 6. AMAZON SNS

## 6.1 Core Purpose

**Problem solved:** Managed pub/sub messaging — one producer publishes to a topic, multiple heterogeneous subscribers receive it simultaneously, eliminating the need to maintain message fan-out logic, subscriber registries, and delivery retry across endpoints.

---

## 6.2 Internal Architecture

```
SNS Architecture:

Publisher → SNS Topic (regional, replicated across AZs)
               │
    ┌──────────┼──────────┬──────────────┐
    │          │          │              │
   SQS       Lambda    HTTPS         Email/SMS
  Queue      Function  Endpoint      (external)
    │          │          │
 Consumer  Processes  Your API
```

### Delivery Internals

```
1. Publish API call received by SNS frontend
2. Message stored in SNS's internal durable log (multi-AZ)
3. SNS delivery subsystem fans out to each subscription endpoint:
   - For each subscription: spawn delivery attempt
   - HTTPS: HTTP POST to subscriber URL
   - Lambda: Invoke API call (async)
   - SQS: SendMessage API call
   - Email: third-party SMTP (no SLA)
4. Each delivery tracked independently:
   - Success: mark delivered
   - Failure: retry with exponential backoff
     - Immediate: 3 retries
     - Pre-backoff: 2 retries (delay 1s)
     - Post-backoff: up to 10 additional retries (delay 20s–600s)
5. Final failure → DLQ (if configured per subscription)
```

### Message Filtering (Server-Side)

```javascript
// Subscription filter policy (JSON, set on subscription)
{
  "eventType": ["order_created", "order_shipped"],
  "amount": [{ "numeric": [">", 100] }],
  "region": [{ "prefix": "us-" }]
}
// SNS evaluates filter against MessageAttributes BEFORE delivery
// Filtered messages NOT billed to subscriber
```

This is evaluated in the SNS delivery subsystem before dispatching to the subscriber endpoint — critical for cost optimization in high-throughput fan-out scenarios.

---

## 6.3 Invocation & Communication Model

- **Push model:** SNS pushes to subscribers (unlike SQS pull)
- **Async:** Publisher gets immediate acknowledgment; delivery is async
- **At-least-once delivery**
- **No backpressure:** SNS will keep trying to deliver; slow subscribers may experience delays

---

## 6.4 Deep-Dive Scenario

```
Fan-out pattern: Single order event → multiple downstream systems

OrderService
    │
    ▼
SNS Topic: order-events
    │
    ├── SQS: inventory-updates (filter: eventType=order_created)
    │       └── Lambda: inventory-reserver
    │
    ├── SQS: fulfillment-queue (filter: eventType=order_created, payment=completed)
    │       └── ECS Task: fulfillment-worker
    │
    ├── Lambda: analytics-ingestor (no filter, all events)
    │
    └── HTTPS: external-webhook (filter: eventType=order_shipped)
```

```javascript
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const sns = new SNSClient({ region: 'us-east-1' });

const publishOrderEvent = async (order, eventType) => {
  await sns.send(new PublishCommand({
    TopicArn: process.env.ORDER_EVENTS_TOPIC_ARN,
    Message: JSON.stringify(order),
    Subject: `Order Event: ${eventType}`,
    MessageAttributes: {
      eventType: { DataType: 'String', StringValue: eventType },
      amount: { DataType: 'Number', StringValue: String(order.amount) },
      region: { DataType: 'String', StringValue: order.region },
    }
  }));
};
```

---

## 6.5 When to Use vs When NOT to Use

**Use:** Fan-out, decoupled event broadcasting, cross-account notifications, mobile push  
**Don't use:** Ordered delivery, message replay, consumer-controlled rate (use Kinesis/SQS instead)

## 6.6 Common Pitfalls

1. **SNS → Lambda without SQS buffer:** SNS invokes Lambda directly. If Lambda throttled → delivery fails → retries → but your Lambda is still throttled. **Fix:** SNS → SQS → Lambda ESM for controlled throughput.
2. **HTTPS endpoint down:** SNS retries but eventually gives up (no infinite retry). Must configure DLQ per subscription.
3. **Large message payloads:** SNS max message size = 256KB. Use SNS Extended Client (S3 + SNS reference).
4. **No message filtering = cost amplifier:** Every subscriber Lambda gets every message → billed per invocation. Add filter policies.

---

# 7. AWS STEP FUNCTIONS

## 7.1 Core Purpose

**Problem solved:** Orchestrating multi-step distributed workflows with state management, error handling, and retry logic — eliminating ad-hoc workflow orchestration code scattered across Lambda functions.

**What breaks:** You chain Lambda → Lambda with SQS between each step, manually track state in DynamoDB, implement timeout + retry per step in each Lambda, and end up with a "distributed spaghetti" that's impossible to debug.

---

## 7.2 Internal Architecture

### State Machine Execution Model

```
State Machine Definition (ASL - Amazon States Language):
  JSON document defining:
  - States (Task, Choice, Wait, Parallel, Map, Pass, Succeed, Fail)
  - Transitions between states
  - Error handling (Catch/Retry)
  - Input/output processing (InputPath, OutputPath, ResultPath, Parameters)

Execution:
  Each execution = isolated, persistent state machine run
  State = stored in Step Functions' durable state store (NOT in Lambda memory)
  
Internal components:
┌──────────────────────────────────────────────────────────┐
│ CONTROL PLANE                                             │
│  - State Machine registry (CreateStateMachine API)        │
│  - Execution scheduler                                    │
│  - Workflow engine (evaluates ASL)                        │
└──────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────┐
│ DATA PLANE                                                │
│  - Execution state store (current state, input/output)    │
│  - Activity task queue (for Activity workers)             │
│  - Event history log (all transitions)                    │
└──────────────────────────────────────────────────────────┘
```

### State Transition Internals

```
1. StartExecution called → execution ID assigned
2. Workflow engine reads state machine definition
3. Current state = StartAt state
4. For TASK state:
   a. Worker identifies integration type:
      - Lambda: synchronous Invoke API call
      - SQS: SendMessage API call
      - DynamoDB: PutItem/GetItem API calls
      - HTTP (Express only): direct HTTPS call
   b. Task heartbeat tracking (HeartbeatSeconds): if task doesn't respond, TimeoutError
   c. Wait for response (Standard) or fire-and-forget (Express async integrations)
5. Response processed through ResultPath/OutputPath
6. Transitions to next state (based on Next field or End: true)
7. For CHOICE state: evaluates conditions against current input JSON
8. For PARALLEL state:
   - All branches start simultaneously
   - Each branch is an independent sub-execution
   - Parent waits for ALL branches to complete
   - If any branch fails: all branches cancelled (by default)
9. For MAP state:
   - Array of items → one iteration per item (parallel, configurable MaxConcurrency)
   - Each iteration is independent
10. State + history stored after each transition
```

### How Step Functions Maintains State

This is the key differentiator:

```
Standard Workflows:
  - State persisted to durable store after EVERY state transition
  - Can survive regional event (state restored on recovery)
  - Max duration: 1 year
  - Execution history: up to 25,000 events per execution
  - Billed per state transition ($0.025/1,000)

Express Workflows:
  - State in memory + async logged to CloudWatch Logs
  - Max duration: 5 minutes
  - No execution history query (must use CloudWatch)
  - Billed per execution + duration ($1.00/million + $0.00001/GB-second)
  - Designed for high-throughput event processing
```

### Retry and Error Handling Internals

```json
{
  "Retry": [
    {
      "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
      "IntervalSeconds": 2,
      "MaxAttempts": 6,
      "BackoffRate": 2,
      "JitterStrategy": "FULL"
    }
  ],
  "Catch": [
    {
      "ErrorEquals": ["States.ALL"],
      "Next": "HandleError",
      "ResultPath": "$.error"
    }
  ]
}
```

Internal retry: Step Functions workflow engine manages retry counter in execution state. After max attempts → falls through to Catch. JitterStrategy=FULL adds full jitter to prevent thundering herd on retry storms.

---

## 7.3 Deep-Dive Scenario (Node.js)

```javascript
// State machine definition (ASL) for order processing
const definition = {
  Comment: "Order processing workflow",
  StartAt: "ValidateOrder",
  States: {
    ValidateOrder: {
      Type: "Task",
      Resource: "arn:aws:lambda:us-east-1:123456789:function:validate-order",
      TimeoutSeconds: 10,
      Retry: [{ ErrorEquals: ["Lambda.TooManyRequestsException"], 
                IntervalSeconds: 1, MaxAttempts: 3, BackoffRate: 2 }],
      Catch: [{ ErrorEquals: ["ValidationError"], Next: "OrderRejected" }],
      Next: "ProcessPayment"
    },
    ProcessPayment: {
      Type: "Task",
      Resource: "arn:aws:states:::lambda:invoke.waitForTaskToken",
      // waitForTaskToken = Lambda gets token, calls SendTaskSuccess externally
      // Useful for async payment gateway callbacks
      Parameters: {
        FunctionName: "process-payment",
        "Payload.$": "$",
        "TaskToken.$": "$$.Task.Token"  // Step Functions injects token
      },
      HeartbeatSeconds: 300,  // fail if no heartbeat for 5 min
      Next: "FulfillOrder"
    },
    FulfillOrder: {
      Type: "Parallel",
      Branches: [
        {
          StartAt: "UpdateInventory",
          States: {
            UpdateInventory: {
              Type: "Task",
              Resource: "arn:aws:lambda:...:function:update-inventory",
              End: true
            }
          }
        },
        {
          StartAt: "SendConfirmation",
          States: {
            SendConfirmation: {
              Type: "Task",
              Resource: "arn:aws:states:::sns:publish",
              Parameters: {
                TopicArn: "arn:aws:sns:...:order-notifications",
                "Message.$": "States.Format('Order {} confirmed', $.orderId)"
              },
              End: true
            }
          }
        }
      ],
      Next: "OrderComplete"
    },
    OrderComplete: { Type: "Succeed" },
    OrderRejected: { Type: "Fail", Error: "OrderRejected", Cause: "Validation failed" }
  }
};

// waitForTaskToken pattern in the payment Lambda:
const processPayment = async (event) => {
  const { orderId, amount, taskToken } = event;
  
  // Call external payment gateway asynchronously
  await paymentGateway.initiatePayment({
    orderId,
    amount,
    callbackData: { taskToken }  // store token for callback
  });
  
  // Payment gateway will call our webhook, which calls:
  // sfn.sendTaskSuccess({ taskToken, output: JSON.stringify({paymentId: 'py_123'}) })
  // This RESUMES the Step Functions execution at the ProcessPayment state
};
```

---

## 7.4 When to Use

| Use When | Tradeoff |
|----------|---------|
| Multi-step workflows with error handling | Per-state transition cost for Standard |
| Long-running async orchestration (weeks/months) | Complex ASL learning curve |
| Parallel fan-out coordination | Express Workflows needed for high-throughput |
| Human approval workflows (waitForTaskToken) | |
| Saga pattern (distributed transactions with compensation) | |

## 7.5 When NOT to Use

- **High-throughput event processing (>1k/sec Standard):** Use Express Workflows or pure Lambda/SQS
- **Simple 2-step workflows:** Overhead of Step Functions not worth it; use SQS + Lambda
- **Real-time sub-second workflows:** Step Functions has ~100ms overhead per state transition

## 7.6 Common Pitfalls

1. **Payload size limit:** Input/output per state = 256KB. Large results must be stored in S3 and the S3 key passed between states.
2. **Task Token expiry:** waitForTaskToken tokens don't expire explicitly, but execution can timeout. If callback never comes → execution stuck until ExecutionTimeout.
3. **Parallel state failure:** Any branch failure cancels ALL branches (default). You may want Catch inside each branch to handle failures gracefully.
4. **History limit:** Standard Workflow max 25,000 events. A Map state iterating 10,000 items generates 4+ events per item = exceeds limit. Use Express or limit iteration.

---

# 8. AMAZON CLOUDFRONT

## 8.1 Core Purpose

**Problem solved:** Content Delivery Network — caches content at 450+ global edge locations, reducing latency for end users and offloading origin servers from repeated identical requests.

---

## 8.2 Internal Architecture

### Edge Location Hierarchy

```
CloudFront Network (3-tier):

TIER 1: Edge Locations (450+ globally)
  - Closest to end users
  - Serve cached content
  - L1 cache (fast SSD)
  - ~1ms to major cities

TIER 2: Regional Edge Caches (13 globally)
  - Larger cache, longer TTL
  - Sits between Edge Locations and Origin
  - L2 cache (larger HDD + SSD)
  - If L1 miss: check L2 before going to origin

TIER 3: Origin
  - Your S3 bucket, ALB, EC2, API GW, custom HTTP
```

### Request Routing (BGP Anycast)

```
How CloudFront routes user to nearest edge:
1. CloudFront's ASN announces the same IP range from ALL edge locations
2. Internet BGP routing selects nearest edge by AS-path length + latency
3. DNS resolution: CloudFront's authoritative DNS returns edge IP
   (actually: user's DNS resolver → CloudFront DNS → returns IP of nearest edge)
4. User connects to that edge location
5. TLS terminated at edge (certificate stored at edge)
6. Request processed at edge
```

### Cache Key and Cache Behavior

```
Cache Key (what makes two requests "the same"):
  Default cache key = URL + Host header

Customizable cache key:
  - Query strings (include/exclude/specific)
  - Headers (include/exclude/specific)
  - Cookies (include/exclude/specific)
  - Compression (Accept-Encoding normalized)

CRITICAL: Adding headers/cookies to cache key fragments cache
  Example: Cache key includes User-Agent → separate cache entry per browser
  → Cache hit rate plummets → more origin requests → defeats purpose
```

### Cache Invalidation Internals

```
CreateInvalidation API call:
  1. Invalidation request sent to CloudFront control plane
  2. Control plane propagates invalidation to ALL edge locations
  3. Propagation delay: up to 15 minutes (most are ~1-3 minutes)
  4. Each edge marks cached copy stale
  5. Next request to that path: cache miss → fetch from origin
  6. Invalidation cost: $0.005/path (first 1,000/month free)

Wildcards: /images/* invalidates all paths under /images/
  - Counted as 1 invalidation per wildcard path (not per file)
  
Alternative to invalidation: Versioned URLs
  /app.v2.3.1.js instead of /app.js
  → Old file stays in cache (harmless)
  → New file has new URL → no cache hit needed
  → Zero cost, instant "invalidation" via URL change
  → PREFERRED over API invalidation in production
```

### Origin Fetch Flow

```
1. Edge L1 cache MISS
2. Edge checks Regional Edge Cache (L2)
   L2 HIT → return to edge, cache at L1, serve user
   L2 MISS → 
3. Origin Request:
   a. CloudFront edge → TCP connection to origin (reused connection pool)
   b. If origin = S3: request includes Origin Access Control (OAC) SigV4 header
   c. If origin = ALB/EC2: request includes X-Forwarded-For, CloudFront headers
   d. Origin returns response with Cache-Control header
4. Response cached at L2 (if Vary + Cache-Control allow it)
5. Response cached at L1 edge
6. Served to user

Cache-Control headers (from origin) control caching:
  max-age=86400          → cache 1 day at edge
  s-maxage=3600          → edge caches 1 hour (overrides max-age for shared caches)
  no-cache               → revalidate with origin before each serve
  no-store               → never cache
  private                → user-specific, don't cache at edge
```

### Lambda@Edge and CloudFront Functions

```
Lambda@Edge (runs on regional edge):
  - Viewer Request: before cache check
  - Origin Request: on cache miss, before going to origin
  - Origin Response: after origin response, before caching
  - Viewer Response: before returning to user
  - Node.js 18 / Python 3.x
  - 5-second timeout (viewer) / 30-second (origin)
  - Replicated to ALL edge PoPs automatically

CloudFront Functions (runs on edge):
  - Viewer Request / Viewer Response only
  - JavaScript (subset of JS)
  - <1ms execution time
  - 10KB max code size
  - 2MB/s max memory
  - Use for: URL rewrites, header manipulation, simple auth
  - 1/6 the cost of Lambda@Edge
```

---

## 8.3 Deep-Dive Scenario

```
SPA + API + Assets Architecture:

Browser → CloudFront Distribution
            │
  ┌─────────┼──────────────────────────────────┐
  │         │                                   │
  │   ┌─────▼─────────────────────────┐         │
  │   │ Cache Behavior 1: /api/*       │         │
  │   │ - TTL: 0 (no cache)           │         │
  │   │ - Origin: ALB (api-alb.aws)   │         │
  │   │ - Forward: All headers, cookies│         │
  │   └───────────────────────────────┘         │
  │                                             │
  │   ┌─────────────────────────────────┐       │
  │   │ Cache Behavior 2: /static/*     │       │
  │   │ - TTL: 31536000 (1 year)        │       │
  │   │ - Cache key: path only          │       │
  │   │ - Origin: S3 bucket             │       │
  │   │ - Compress: true                │       │
  │   └─────────────────────────────────┘       │
  │                                             │
  │   ┌─────────────────────────────────┐       │
  │   │ Default Behavior: /*            │       │
  │   │ - TTL: 0 (for SPA HTML)         │       │
  │   │ - CloudFront Function:          │       │
  │   │   - Rewrite /app/* → /index.html│       │
  │   │ - Origin: S3 bucket             │       │
  │   └─────────────────────────────────┘       │
  └─────────────────────────────────────────────┘
```

---

## 8.4 When to Use vs Not Use

**Use:** Static assets, SPAs, media delivery, global API acceleration, DDoS mitigation (AWS Shield integrated), geo-restriction  
**Don't use:** WebSocket APIs (CloudFront has 1-minute idle connection timeout → use ALB directly), highly personalized content (cache hit rate = 0)

## 8.5 Common Pitfalls

1. **Caching API responses with auth cookies/headers:** Without configuring cache key correctly, User A's API response served to User B.
2. **Missing Origin Access Control:** S3 bucket public because team forgot to restrict direct access. OAC ensures only CloudFront can access S3.
3. **Cache invalidation in CI/CD:** `aws cloudfront create-invalidation --paths "/*"` on every deploy = expensive + slow. Use versioned filenames instead.
4. **HTTPS to origin:** CloudFront terminates TLS from user. If origin is HTTP, data CloudFront→origin is unencrypted. Use Origin Protocol Policy: HTTPS Only.

---

# 9. APPLICATION LOAD BALANCER (ALB)

## 9.1 Core Purpose

**Problem solved:** Layer 7 (HTTP/HTTPS/WebSocket/gRPC) load balancing with content-based routing, sticky sessions, and native integration with AWS services — eliminating custom nginx/HAProxy management.

---

## 9.2 Internal Architecture

### Control Plane vs Data Plane

```
CONTROL PLANE:
  - ALB fleet management
  - Node provisioning (ALB scales by adding nodes)
  - Health check coordination
  - Target group registration

DATA PLANE:
  - ALB nodes (EC2-like instances, managed by AWS)
  - Multiple nodes per AZ (scales automatically)
  - Each node handles subset of connections
  - DNS round-robins across all nodes
```

### Request Lifecycle

```
1. DNS lookup → returns multiple IPs (one per ALB node per AZ)
2. Client connects to one IP (one ALB node)
3. ALB node: TLS termination (certificate from ACM)
4. HTTP parsing: method, path, headers, query string
5. Listener rule evaluation (in priority order):
   - Host header match: api.example.com → target group A
   - Path pattern: /api/v2/* → target group B
   - Query string: ?version=2 → target group C
   - Header: X-API-Version: 2 → target group C
   - Default: target group D
6. Target selection within target group:
   - Algorithm: Round Robin (HTTP/HTTPS) or Least Outstanding Requests
   - Sticky sessions: hash of cookie → same target (source-based)
7. ALB node → target (EC2, ECS, Lambda, IP)
   - To EC2/ECS: reuses TCP connection from connection pool
   - To Lambda: Invoke API call
8. Response: ALB adds X-Amzn-Trace-Id header (X-Ray tracing)
9. Response returned to client over original connection
```

### ALB Scaling Internals

```
ALB does NOT expose its node count. Internally:
1. ALB monitors: connections/sec, new connections/sec, active connections
2. When load increases → pre-warming not available in real-time
   → ALB adds new ALB nodes (takes ~1-7 minutes)
   → DNS TTL (60s) → new clients route to new nodes
   
Pre-warming: If you know traffic spike coming (e.g., event/sale):
  → Contact AWS Support to pre-warm ALB
  → Without pre-warming: 50% traffic surge may overwhelm ALB before auto-scale

Cross-zone load balancing:
  - Default ON: each node balances across all AZs
  - OFF: each node only routes to targets in same AZ
  - ON is recommended for uneven target distribution
```

---

## 9.3 Communication Model

- **Push:** Clients push requests to ALB
- **Synchronous:** Client holds TCP connection
- **Request-driven:** HTTP request/response
- **Backpressure:** ALB queue has a limit; when maxconn reached, returns 503

---

## 9.4 Deep-Dive Scenario (Blue/Green Deployment)

```
ALB with weighted routing for blue/green:

Listener Rule:
  Path: /api/* → Forward to:
    Target Group: BLUE  (weight: 90)
    Target Group: GREEN (weight: 10)  ← canary

Gradually shift: 90/10 → 50/50 → 0/100
If GREEN healthy → remove BLUE targets
```

---

## 9.5 When to Use vs Not Use

**Use:** Microservices routing (host/path-based), WebSocket, gRPC, Lambda targets, HTTP/2  
**Don't use:** UDP traffic, raw TCP (use NLB), sub-millisecond latency requirements

## 9.6 Common Pitfalls

1. **Idle connection timeout:** ALB default = 60 seconds. Keep-alive on targets must be > 60s or connections dropped mid-request.
2. **Lambda targets and concurrency:** ALB → Lambda doesn't respect Lambda reserved concurrency. Throttled Lambda returns 502 to ALB.
3. **Health check sensitivity:** Health check interval too aggressive → auto-scaling group terminates healthy instances. Set grace period > startup time.
4. **400 errors from body size:** ALB max request body = unlimited BUT Lambda integration max = 1MB (base64 encoded 6MB).

---

# 10. NETWORK LOAD BALANCER (NLB)

## 10.1 Core Purpose

**Problem solved:** Layer 4 (TCP/UDP/TLS) load balancing with ultra-low latency and static IP support — for workloads that need raw TCP passthrough or protocols ALB doesn't understand.

---

## 10.2 Internal Architecture

### NLB vs ALB Key Differences

```
ALB: Layer 7
  - Terminates HTTP/HTTPS connections
  - Reads headers, body, cookies
  - Returns responses on behalf of targets
  - Adds latency (~1-2ms) for HTTP parsing
  - Connection: Client ↔ ALB | ALB ↔ Target (two TCP connections)

NLB: Layer 4
  - Passes TCP packets directly (no HTTP parsing)
  - Preserves source IP (no X-Forwarded-For needed)
  - Connection: Client ↔ Target (one TCP connection, NLB is transparent)
  - Latency: ~100μs (microseconds!)
  - One static IP per AZ (Elastic IP assignable)
```

### NLB Request Flow

```
1. Client → NLB IP (TCP SYN)
2. NLB hashing: hash(src-IP, src-port, dst-IP, dst-port, protocol) → target
   - Same 5-tuple = always same target (connection persistence)
3. NLB modifies packet destination IP → target IP (DNAT)
4. Packet forwarded to target
5. Target responds directly to client (DSR - Direct Server Return)
   OR via NLB depending on configuration
6. NLB tracks connection state in flow table (stateful packet inspection)
```

### Zonal Isolation

NLB nodes are per-AZ. DNS returns one IP per AZ. Client connects to AZ-specific NLB node. This provides strict zonal isolation — useful for financial workloads requiring AZ-level failure domain guarantees.

---

## 10.3 When to Use vs Not Use

**Use:** gRPC (with HTTPS passthrough), gaming UDP traffic, IoT MQTT, database connections (static IP for whitelisting), PrivateLink endpoints  
**Don't use:** HTTP routing by path/host (use ALB), Lambda targets (not supported), WebSocket (ALB handles better)

## 10.4 Common Pitfalls

1. **Security groups:** NLB preserves source IP. Target's security group must allow client IPs (not just NLB IPs). ALB: allow only ALB SG.
2. **TLS termination:** NLB can terminate TLS or pass through. Pass-through = target handles TLS = can see original client cert. Termination = NLB strips TLS.
3. **Health check sensitivity:** NLB defaults to TCP health check (just port open). Use HTTP/HTTPS health check to check application health.

---

# 11. AMAZON VPC

## 11.1 Core Purpose

**Problem solved:** Isolated virtual network within AWS with full control over IP addressing, subnets, routing, and network access controls — logical data center in the cloud.

---

## 11.2 Internal Architecture

### VPC Network Stack

```
AWS Physical Network
       │
   Hypervisor (Nitro)
       │
   ┌───▼──────────────────────────────────┐
   │  VPC (logical network boundary)       │
   │                                      │
   │  ┌────────────┐  ┌────────────┐      │
   │  │ Public Sub │  │ Private Sub│      │
   │  │ 10.0.1.0/24│  │ 10.0.2.0/24│     │
   │  └─────┬──────┘  └─────┬──────┘     │
   │        │               │            │
   │  ┌─────▼──────────────▼──────┐      │
   │  │    Route Table              │     │
   │  │  0.0.0.0/0 → IGW            │     │
   │  │  10.0.0.0/16 → local        │     │
   │  └─────────────────────────────┘     │
   │        │                            │
   │  ┌─────▼──────┐                     │
   │  │ Internet GW│                     │
   │  └────────────┘                     │
   └──────────────────────────────────────┘
```

### Nitro Network Virtualization

VPC networking is implemented in the **AWS Nitro System** (not in the guest OS):
- Each EC2 instance runs on a Nitro hypervisor
- VPC packet processing happens in Nitro Card hardware (not CPU)
- Route table lookups happen in Nitro fabric, not in the guest OS kernel
- Security groups are implemented as stateful packet filters in Nitro (not iptables)
- This is why security groups scale to millions of rules without CPU overhead

### Subnet, ENI, and IP Assignment

```
ENI (Elastic Network Interface):
  - Virtual NIC attached to EC2/Lambda/ECS task
  - Has primary private IPv4 + optional secondary IPs
  - MAC address bound to ENI (not instance — IP moves with ENI)
  - Security groups attached to ENI (not subnet or instance)
  - Lambda VPC mode: uses "hyperplane ENI" (shared across Lambda sandboxes,
    managed by Lambda VPC Gateway, not per-function)

IP flow:
  EC2 instance requests DHCP → VPC DHCP server assigns IP from subnet CIDR
  NAT Gateway: private IP → Elastic IP for outbound internet (SNAT)
  Public IP: IGW maintains 1:1 mapping (EC2 never sees public IP in OS)
```

### VPC Peering vs Transit Gateway

```
VPC Peering:
  - Direct 1:1 route between two VPCs
  - No transitive routing (A↔B, B↔C does NOT mean A↔C)
  - Max 125 peering connections per VPC
  - Zero cost for data transfer in same region

Transit Gateway:
  - Central hub routing (star topology)
  - Transitive routing supported
  - Max 5,000 VPC attachments
  - $0.05/GB data transfer processed
  - $0.05/attachment-hour
  - Use when N VPCs need full mesh (>10 VPCs → TGW cheaper than N² peerings)
```

---

## 11.3 Common Pitfalls

1. **Lambda in VPC without NAT:** Lambda in private subnet → no internet access → SDK calls to DynamoDB/S3 fail unless using VPC Endpoints or NAT Gateway.
2. **VPC Endpoint missing:** Traffic to DynamoDB/S3 goes over NAT Gateway ($0.045/GB) when VPC Gateway Endpoint is free.
3. **Security group circular dependency:** SG-A allows SG-B, SG-B allows SG-A → fine. But CloudFormation circular dependency in creation order → stack rollback.
4. **CIDR overlap:** Peered VPCs must not overlap CIDRs. Plan CIDR ranges at org level using AWS RAM.

---

# 12. AMAZON ECS

## 12.1 Core Purpose

**Problem solved:** Container orchestration — scheduling Docker containers across a fleet of EC2 instances (or Fargate), managing placement, health, service discovery, and rolling deployments.

---

## 12.2 Internal Architecture

### ECS Components

```
ECS Control Plane (AWS managed):
  ┌─────────────────────────────────────────────────────┐
  │ ECS API (CreateService, RunTask, etc.)               │
  │ Cluster State Manager                                │
  │ Scheduler (placement decisions)                      │
  │ Service Scheduler (desired count reconciliation)     │
  └─────────────────────────────────────────────────────┘
           │
           ▼
ECS Data Plane (your resources):
  ┌─────────────────────────────────────────────────────┐
  │ EC2 Instances (Container Instances)                  │
  │  └─ ECS Agent (Go binary, talks to ECS API)         │
  │  └─ Docker Engine (containerd)                      │
  │  └─ Task containers                                 │
  └─────────────────────────────────────────────────────┘
```

### ECS Agent Internals

ECS Agent is a Go binary running on each EC2 container instance:
- Polls ECS control plane for tasks to start/stop (long polling)
- Manages container lifecycle via Docker/containerd API
- Reports resource availability (CPU/memory) to ECS scheduler
- Handles task health reporting
- Manages IAM role credential rotation for tasks (IMDS proxy)

### Task Scheduler (Placement)

```
When RunTask or Service scaling is triggered:
1. ECS Scheduler receives task definition
2. Evaluates placement constraints:
   - distinctInstance: don't place two tasks on same instance
   - memberOf: only on instances with specific attributes (instance type, AZ)
3. Evaluates placement strategies:
   - binpack: fill one instance before using next (cost-efficient)
   - spread: distribute across AZs/instances (fault-tolerant)
   - random: random instance selection
4. Filters instances: must have sufficient CPU + memory
5. Selects instance, sends StartTask to ECS Agent
6. Agent pulls Docker image → starts container
```

### Service Scheduler (Long-running services)

```
Desired count = 3, Running count = 2:
  → Service Scheduler detects delta
  → Calls RunTask to start 1 more task
  → Task placed on available instance
  → ALB target group registration (if configured)
  → Health check passes → service healthy

Rolling update (ECS deployment):
  minimumHealthyPercent: 50%  → can kill 50% old tasks
  maximumPercent: 200%        → can run 200% of desired (spare capacity)
  
  Process:
  1. Start N new tasks (new task def revision)
  2. Wait for new tasks healthy
  3. Stop N old tasks
  4. Repeat until all replaced
```

---

## 12.3 Deep-Dive Scenario (Node.js Service)

```
ECS Service: payment-processor
  - Service type: Replica (desired: 3)
  - Launch type: EC2
  - Task definition: 
      cpu: 512, memory: 1024
      image: 123456789.dkr.ecr.us-east-1.amazonaws.com/payment:v2.3
      environment variables from SSM Parameter Store
      logging: awslogs to CloudWatch
  - ALB target group: port 3000
  - Service Connect: service-to-service discovery
  
Auto-scaling:
  - Target tracking: ALBRequestCountPerTarget = 1000
  - Step scaling: SQS queue depth > 1000 → scale out
```

---

# 13. AWS FARGATE

## 13.1 Core Purpose

**Problem solved:** Serverless containers — run containers without managing EC2 instances, eliminating cluster capacity management, OS patching, and instance type selection.

---

## 13.2 Internal Architecture

### Fargate vs EC2 Launch Type — The Real Difference

```
EC2 Launch Type:
  Your EC2 instances → ECS Agent → pull tasks onto YOUR instances
  You manage: instance type, AMI, patching, cluster capacity

Fargate:
  AWS manages the underlying EC2 instances (invisible to you)
  Each TASK gets its own micro-VM (via Firecracker, similar to Lambda)
  Task isolation: hardware VM boundary (not just container namespace)
  You see: tasks only, not instances
  
Fargate Task Lifecycle:
1. RunTask called
2. Fargate control plane allocates a Firecracker MicroVM
   (sized to your task CPU/memory specification)
3. Container images pulled (from ECR, Docker Hub)
4. Containers started within the MicroVM
5. Task runs; ENI attached to your VPC subnet
6. Task completes or is stopped → MicroVM released
```

### Fargate ENI per Task

Unlike EC2 (shared ENI), each Fargate task gets its own ENI:
- Own IP address in your subnet
- Security group per task
- AWS manages ENI lifecycle
- **Implication:** Large Fargate services consume many IPs from your subnet CIDR

### Fargate CPU/Memory Combinations

```
Fargate vCPU → Memory matrix:
0.25 vCPU: 0.5GB, 1GB, 2GB
0.5  vCPU: 1GB–4GB  
1    vCPU: 2GB–8GB
2    vCPU: 4GB–16GB
4    vCPU: 8GB–30GB
16   vCPU: 32GB–120GB (Fargate for compute-intensive)

Granularity: You pay for exact allocated CPU/memory, not instance types
```

---

## 13.3 When to Use Fargate vs ECS/EC2

| | Fargate | ECS EC2 |
|-|---------|---------|
| Cluster management | None (AWS) | You manage instances |
| Startup time | 30–90s (image pull) | 30–90s (if instance warm) |
| Isolation | MicroVM per task | Container per instance (shared kernel) |
| Cost model | Per second (vCPU + memory) | Instance-hour + container overhead |
| Spot support | Fargate Spot (70% discount, can be reclaimed) | EC2 Spot |
| Custom runtime | No | Yes (any AMI) |
| GPU workloads | No | Yes (GPU instances) |
| Windows containers | Yes | Yes |
| Sustained high-density | More expensive than EC2 | Cost-efficient at high density |

---

# 14. AMAZON EC2

## 14.1 Core Purpose

**Problem solved:** Virtual machines in the cloud — resizable compute instances with full OS control, persistent state, and flexible networking.

---

## 14.2 Internal Architecture

### Nitro System

```
Physical Server
└── Nitro Hypervisor (KVM-based, runs in hardware)
    ├── Nitro Card (Network): VPC routing, security groups
    ├── Nitro Card (Storage): EBS NVMe access
    ├── Nitro Card (Controller): Monitoring, management
    └── Guest VM (your EC2 instance)
        └── OS → sees: NVMe disk, ENA network interface
```

**Why Nitro matters:**
- Compute: No hypervisor overhead (bare-metal performance for Nitro instances)
- Network: Line-rate network processing in hardware (not CPU)
- Storage: NVMe-style direct I/O to EBS (not through hypervisor path)

### EC2 Instance Types — Hardware Reality

```
Instance families:
  m-series: balanced (multi-tenant, Nitro)
  c-series: compute-optimized (higher clock speed)
  r-series: memory-optimized (more DRAM)
  i-series: storage-optimized (local NVMe SSDs)
  p/g-series: GPU instances (NVIDIA hardware, pass-through via Nitro)
  metal: bare metal (no hypervisor, direct hardware access)
  
Placement Groups:
  Cluster: same rack → ultra-low latency between instances (HPC)
  Spread: different racks → failure isolation (7 instances per AZ per group)
  Partition: different partitions → rack-level isolation at scale (Kafka, Cassandra)
```

### EBS Internals

```
EBS volume:
  - Network-attached block storage (not local disk)
  - Replicated within AZ (3 copies)
  - Accessed via Nitro NVMe card (not through hypervisor)
  
io2 Block Express:
  - Sub-millisecond latency (256μs)
  - 256,000 IOPS max per volume
  - 4TB–64TB
  
gp3:
  - 3,000 IOPS baseline (free), up to 16,000 IOPS
  - 125 MB/s throughput baseline, up to 1,000 MB/s
  - Cheaper than io2 for moderate workloads
```

---

## 14.3 When to Use EC2

**Use:** Long-running services, stateful workloads (databases), GPU/ML training, Windows Server, custom OS requirements, HPC clusters, legacy applications requiring persistent filesystem

**Don't use:** Short-lived jobs (Lambda), fully managed services available (use RDS not self-managed MySQL), pure event-driven (Lambda/Fargate more cost-efficient)

---

# 15. AMAZON CLOUDWATCH

## 15.1 Core Purpose

**Problem solved:** Centralized observability platform for metrics, logs, alarms, and dashboards across all AWS services and custom workloads — eliminating the need to run your own Prometheus/ELK/Grafana for AWS-native workloads.

---

## 15.2 Internal Architecture

### Metrics Internals

```
CloudWatch Metric Flow:

AWS Services → CloudWatch Agent → Metrics API
                                        │
                              ┌─────────▼──────────────┐
                              │  Metrics Storage         │
                              │  Time-series database    │
                              │  Partitioned by:         │
                              │   - Namespace            │
                              │   - MetricName           │
                              │   - Dimensions           │
                              │  Stored in S3-backed     │
                              │  columnar store          │
                              └──────────────────────────┘

Resolution:
  Standard: 1-minute granularity (free), retained 15 months
  High-resolution: 1-second granularity ($0.30/metric-month extra)
  
Aggregation:
  Statistics: Sum, Average, Min, Max, SampleCount, Percentile (p99, p95)
  Extended statistics: percentiles computed at aggregation time
  
Retention:
  <60s: 3 hours
  1min: 15 days
  5min: 63 days
  1hour: 455 days (15 months)
```

### Alarms Internals

```
Alarm evaluation:
1. CloudWatch evaluates metric every period (60s or custom)
2. M of N datapoints threshold:
   - AlarmThreshold: 5 datapoints > 90% CPU in last 10 minutes
   - Missing data: MISSING, IGNORE, BREACHING, NOT_BREACHING (configurable)
3. State transition: OK → ALARM → OK
4. Actions on transition:
   - Auto Scaling Group: scale out/in
   - SNS: publish to topic
   - EC2: stop/terminate/recover
   - Lambda: invoke function (via SNS)
5. Composite alarms: AND/OR of multiple alarms → reduces alarm noise
```

### CloudWatch Logs Internals

```
Log Group → Log Streams → Log Events

Ingestion:
  - CloudWatch Agent on EC2
  - Lambda (automatic, via log agent in MicroVM)
  - ECS (awslogs driver → CW Logs API)
  - CloudTrail, VPC Flow Logs, ALB Access Logs (S3 or CW)

Log Insights (query engine):
  - Behind the scenes: columnar scan of log data in S3-backed store
  - Query syntax similar to SQL
  - Parallel scan across log streams within time window
  - Up to 20 concurrent queries per account
  - Cost: $0.005 per GB scanned

Subscription Filters:
  - Real-time log stream to Lambda, Kinesis, Kinesis Firehose
  - Pattern matching on log events before delivery
  - Max 2 subscription filters per log group
```

---

## 15.3 Deep-Dive Scenario

```javascript
const { CloudWatchClient, PutMetricDataCommand } = require('@aws-sdk/client-cloudwatch');
const cw = new CloudWatchClient({ region: 'us-east-1' });

// Custom business metric
const recordOrderProcessingTime = async (duration, status) => {
  await cw.send(new PutMetricDataCommand({
    Namespace: 'OrderService',
    MetricData: [
      {
        MetricName: 'OrderProcessingDuration',
        Dimensions: [
          { Name: 'Status', Value: status },
          { Name: 'Environment', Value: process.env.ENV }
        ],
        Value: duration,
        Unit: 'Milliseconds',
        Timestamp: new Date(),
        StorageResolution: 1  // high resolution (1-second)
      },
      {
        MetricName: 'OrderCount',
        Dimensions: [{ Name: 'Status', Value: status }],
        Value: 1,
        Unit: 'Count',
      }
    ]
  }));
};
// Use Embedded Metric Format (EMF) in Lambda for zero-latency metric emission:
// console.log(JSON.stringify({
//   "_aws": { "Timestamp": Date.now(), "CloudWatchMetrics": [{
//     "Namespace": "OrderService",
//     "Dimensions": [["Status"]],
//     "Metrics": [{"Name": "OrderCount", "Unit": "Count"}]
//   }]},
//   "Status": "SUCCESS",
//   "OrderCount": 1
// }));
```

## 15.4 Common Pitfalls

1. **PutMetricData throttling:** Max 40 transactions/second; 150 metrics per call. Batch metrics before publishing.
2. **Log Insights cost blindspot:** Querying 30 days of high-volume Lambda logs = multiple TB scanned = unexpected bill.
3. **Alarm evaluation with missing data:** Default = MISSING (alarm doesn't fire). For Lambda: if function never invokes, metric missing → alarm stays OK. Use `treat_missing_data = breaching` for critical metrics.
4. **Metrics resolution gotcha:** High-resolution metrics (1s) retained only 3 hours at 1-second, then rolled to 1-minute. Don't build dashboards expecting 1-second precision on yesterday's data.

---

# 16. AWS X-RAY

## 16.1 Core Purpose

**Problem solved:** Distributed tracing across microservices — correlating requests across Lambda, API GW, ECS, SQS chains into a single trace for latency analysis and failure attribution.

---

## 16.2 Internal Architecture

### Tracing Model

```
Trace = collection of segments from one request
Segment = one service's participation in the request
Subsegment = nested operation within a segment (HTTP call, DynamoDB query)

Trace ID propagated via:
  HTTP: X-Amzn-Trace-Id header
  SQS: MessageAttribute (X-Amzn-Trace-Id)
  SNS: message attribute
  Lambda: environment variable (_X_AMZN_TRACE_ID)

X-Ray Daemon:
  - Lightweight process running alongside your app
  - Receives segments on UDP port 2000 (localhost)
  - Buffers and sends to X-Ray API in batches
  - Buffers up to 50MB in memory
  - 1-second flush interval
  
Lambda X-Ray:
  - X-Ray daemon runs in Lambda sandbox
  - SDK sends segments to daemon via UDP
  - Daemon flushes after invocation completes
```

### Sampling Internals

```
Default sampling rule:
  - First request each second per host: always sampled
  - After that: 5% of requests sampled
  
Custom rules evaluated in priority order:
  - Match on: service name, HTTP method, URL path, resource ARN
  - Set: fixed rate or reservoir (N/second always + M% of remainder)
  
Reservoir = rate limiter to prevent overwhelming X-Ray ingestion
  - Account-level reservoir: 50 traces/second per region
  - If reservoir full: sampling rate applies to all remaining requests
```

### ServiceMap Construction

```
1. Each service publishes segments to X-Ray API
2. X-Ray correlation: trace ID links segments from different services
3. X-Ray service graph built from segment relationships:
   - Caller/callee extracted from segment data
   - Response time, error rate computed per edge
4. ServiceMap = graph visualization of connections + health
```

---

## 16.3 Deep-Dive Scenario (Node.js)

```javascript
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));  // auto-instrument all AWS SDK calls

// Manual subsegment for custom operation
const processOrder = async (order) => {
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('order-validation');
  
  try {
    subsegment.addAnnotation('orderId', order.id);       // searchable
    subsegment.addMetadata('orderDetails', order);        // non-searchable, any JSON
    
    await validateOrder(order);
    subsegment.close();
  } catch (err) {
    subsegment.addError(err);
    subsegment.close();
    throw err;
  }
};

// X-Ray + Express middleware
const app = require('express')();
app.use(AWSXRay.express.openSegment('OrderService'));
// ... routes
app.use(AWSXRay.express.closeSegment());
```

---

## 16.4 Comparison: X-Ray vs OpenTelemetry

| | X-Ray | OpenTelemetry + Jaeger/Zipkin |
|-|-------|-------------------------------|
| AWS integration | Native (Lambda auto-instruments) | Manual setup |
| Vendor lock-in | High | None |
| Cost | $5/million traces | Infrastructure cost |
| ServiceMap | Built-in UI | Build yourself |
| Sampling | AWS-managed | Self-managed |
| gRPC support | Limited | Full |

**AWS Distro for OpenTelemetry (ADOT):** AWS's distribution of OpenTelemetry → sends to X-Ray OR any OTLP backend. Best of both.

---

# 17. SPECIAL MANDATORY DEEP DIVES

## 17.1 Lambda + SQS: Event Source Mapping Internal Flow

This is the most commonly misunderstood Lambda integration. Let's go extremely deep.

### Who Polls SQS?

**The Lambda service itself polls SQS — NOT your Lambda function.**

```
┌───────────────────────────────────────────────────────────────┐
│ LAMBDA SERVICE INTERNAL POLLER FLEET                          │
│                                                               │
│  Poller Process 1 ──────────────────────┐                    │
│  Poller Process 2 ──────────────────────┤──→ SQS Queue       │
│  Poller Process 3 ──────────────────────┘    (ReceiveMessage) │
│  ...up to 5 pollers initially per ESM                        │
└───────────────────────────────────────────────────────────────┘
         │
         │ Batch of messages
         ▼
┌───────────────────────────────────────┐
│ Lambda Worker Manager                  │
│  - Invoke Lambda function with batch   │
│  - Manage concurrency scaling          │
└───────────────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────┐
│ Your Lambda Function (handler)         │
│  Receives: { Records: [...messages] } │
└───────────────────────────────────────┘
```

### ESM Poller Scaling

```
Queue depth scaling algorithm:
  ≤ 1,000 messages: 1–5 pollers (1 poller active to start)
  1,000–10,000 messages: up to 60 pollers
  > 10,000 messages: up to 1,000 pollers (Lambda concurrency limit applies)

Scaling behavior:
  - Lambda ESM polls at 300 short-poll requests/minute initially
  - Detects message backlog → increases pollers
  - Each poller runs ReceiveMessage (long poll, up to 20s wait)
  - Each ReceiveMessage returns up to BatchSize messages
  - Each batch → one Lambda invocation (one concurrent execution)
  
Concurrency math:
  If pollers = 60, each holds 10 messages in batch:
  Up to 60 concurrent Lambda executions
  Each execution processes 10 messages
  Effective throughput: 60 × 10 = 600 messages being processed simultaneously
```

### Batching Internals

```
BatchSize: 1–10 (Standard), 1–10,000 (FIFO)
BatchingWindow (MaximumBatchingWindowInSeconds): 0–300 seconds
  - With window: Lambda waits up to N seconds to accumulate BatchSize messages
  - Reduces Lambda invocations (better for DB writes in bulk)
  - Increases latency for individual messages

When Lambda invoked with batch:
  event.Records = [
    { messageId, receiptHandle, body, attributes, ... },
    { messageId, receiptHandle, body, attributes, ... },
    ...up to BatchSize
  ]

CRITICAL: ALL messages in batch share the same visibility timeout
  - Visibility timeout set when ReceiveMessage called by poller
  - Your Lambda has until timeout expires to process ALL messages in batch
```

### Visibility Timeout + Lambda Processing Time

```
Timeline example (BatchSize=10, VisibilityTimeout=30s):

T=0:  Poller ReceiveMessage → gets 10 messages
      Visibility timeout starts for all 10 messages
      Lambda invoked with 10 messages
T=25: Lambda finishes 8 messages, 2 still in progress
T=30: VISIBILITY TIMEOUT EXPIRES
      SQS makes all 10 messages VISIBLE again
      Another poller receives all 10 messages!
      Lambda completes processing at T=31
      DeleteMessage for completed messages... but receipt handles expired
      → SQS returns error (receipt handle expired) for all deletes
      → ALL 10 messages reprocessed (even the 8 that succeeded)

FIX:
  1. Set VisibilityTimeout = 6× the Lambda function timeout
  2. Or process fast (reduce individual processing time)
  3. ReportBatchItemFailures = return only failed IDs, Lambda ESM handles rest
```

### Partial Batch Response (ReportBatchItemFailures)

```javascript
// Lambda handler with partial batch failure handling
exports.handler = async (event) => {
  const batchItemFailures = [];
  
  await Promise.allSettled(
    event.Records.map(async (record) => {
      try {
        await processMessage(JSON.parse(record.body));
        // Success: Lambda ESM will delete this message automatically
      } catch (err) {
        console.error(`Failed to process ${record.messageId}:`, err);
        batchItemFailures.push({ itemIdentifier: record.messageId });
        // Failure: Lambda ESM will NOT delete this message
        // Message becomes visible again after visibility timeout
        // Receives count incremented → eventually goes to DLQ
      }
    })
  );
  
  // Return failed message IDs
  return { batchItemFailures };
  // Lambda ESM interprets this and only deletes SUCCESSFUL messages
};
```

**Internal mechanics of ReportBatchItemFailures:**
1. Lambda returns `{ batchItemFailures: [{ itemIdentifier: msgId }] }`
2. Lambda ESM service receives this return value
3. ESM calls `DeleteMessageBatch` for all messages NOT in batchItemFailures
4. Failed messages remain in queue with original receipt handle (visible after timeout)
5. Failed messages will be retried up to `maxReceiveCount` times → then DLQ

### DLQ Configuration (Critical to Understand)

```
Two DLQ options — completely different:

1. Lambda DLQ (on function):
   - Only applies to ASYNC invocations (Event invoke type)
   - Does NOT apply to SQS ESM (SQS ESM handles its own retries)
   - Messages from Lambda DLQ are raw invocation events

2. SQS DLQ (on source queue):
   - Applies to ESM polling
   - maxReceiveCount controls how many times message retried
   - After max retries → SQS moves to DLQ automatically
   - DLQ message = original SQS message body
   
For SQS ESM: ALWAYS configure DLQ on the SOURCE SQS QUEUE, not on Lambda function.
```

---

## 17.2 API Gateway → Lambda Internal Flow

### Integration Types

```
1. Lambda Proxy Integration (AWS_PROXY):
   - API GW passes raw HTTP request as JSON event to Lambda
   - Lambda must return structured response: { statusCode, headers, body }
   - No request/response transformation by API GW
   - Most common (used by Serverless Framework, SAM by default)

2. Lambda Non-Proxy Integration (AWS):
   - API GW transforms request using VTL before sending to Lambda
   - Lambda receives transformed payload (not HTTP event)
   - API GW transforms Lambda response using VTL before returning to client
   - Complex to set up, powerful for schema transformation
   
3. HTTP/HTTP_PROXY:
   - API GW proxies request to HTTP endpoint (not Lambda)
   - Proxy: pass-through; Non-proxy: VTL transform
   
4. AWS Service Integration:
   - API GW calls AWS service directly (SQS SendMessage, DynamoDB PutItem)
   - No Lambda involved (saves Lambda cold start + cost)
   - Limited to services supporting API GW integration
   
5. Mock Integration:
   - API GW returns response without calling any backend
   - Useful for OPTIONS CORS preflight, health checks
```

### Detailed Request Flow (Proxy Integration)

```
Client POST /orders
  → API GW frontend fleet
  → Rate limit check (token bucket)
  → Authorizer (if configured)
  → Route match: POST /orders → Lambda: create-order
  → Build Lambda event:
     {
       "version": "1.0",
       "resource": "/orders",
       "path": "/orders",
       "httpMethod": "POST",
       "headers": { "Content-Type": "application/json", ... },
       "queryStringParameters": null,
       "pathParameters": null,
       "body": "{\"item\":\"book\"}",
       "isBase64Encoded": false,
       "requestContext": {
         "requestId": "abc123",
         "stage": "prod",
         "authorizer": { "userId": "u-123" }
       }
     }
  → Lambda Invoke API (InvocationType=RequestResponse)
  → Lambda executes handler, returns:
     {
       "statusCode": 201,
       "headers": { "Content-Type": "application/json" },
       "body": "{\"orderId\":\"o-456\"}"
     }
  → API GW parses response, sets HTTP status and headers
  → Returns HTTP 201 to client
```

### HTTP API vs REST API Internal Differences

```
REST API (v1):
  - VTL transformation engine (Velocity Template Language)
  - Request/response mapping templates stored and evaluated per request
  - Resource-based routing (must define resources and methods)
  - Supports: API Keys, Usage Plans, Request Validation, WAF
  
HTTP API (v2):
  - No VTL engine (simpler internals = lower latency)
  - Route-based routing (simpler config)
  - Auto deploy enabled (no manual deployment needed)
  - JWT authorizer evaluated natively (no Lambda auth overhead)
  - ~70% cheaper, ~7ms less latency
  - Missing: API Keys, Usage Plans, Request/Response transforms, WAF direct integration
```

---

## 17.3 DynamoDB Deep Internals

### Partition Key Hashing (The Math)

```
Internal partition assignment:
  partition_number = floor(MD5(partition_key_value) / (2^128 / num_partitions))
  
Example:
  Table has 4 partitions (10GB table, ~1000 WCU)
  Key "user-123" → MD5 = 0x4A...
  0x4A.../4 → partition 2
  
  Key "user-124" → MD5 = 0x91...
  0x91.../4 → partition 3
  
Keys adjacent in name-space may be in completely different partitions.
Keys that happen to hash similarly will share a partition.
```

### Hot Partition Problem

```
Scenario: All writes have PK = "STATUS" (status-based partition key)
  All writes → hash("STATUS") → same partition
  Partition max throughput: 1,000 WCU
  If table has 10,000 WCU provisioned across 10 partitions:
    Only 1 partition receiving writes → throttled at 1,000 WCU
    9,000 WCU capacity completely wasted

Symptoms:
  - ProvisionedThroughputExceededException on specific items
  - CloudWatch shows consumed WCU << provisioned WCU (aggregate OK but per-partition throttled)
  
Solutions:
  1. High-cardinality PK (userId, orderId — naturally distributed)
  2. Write sharding: PK = userId + "#" + random(1,10)
     → distribute one user's writes across 10 partitions
     → Read all 10 shards and merge in application
  3. Adaptive Capacity (see below)
```

### Adaptive Capacity

```
DynamoDB Adaptive Capacity (automatic):
  - Detects hot partitions in real-time
  - Temporarily boosts that partition's capacity by borrowing from underutilized ones
  - Can boost partition beyond its normal share up to full table capacity
  - NOT infinite: still limited by table-level WCU
  - Delay before adaptation: up to 30 minutes for sustained hot partition
  
Instant Adaptive Capacity (newer, on-demand tables):
  - Adaptation happens in seconds, not minutes
  - Partition automatically isolated and split when hot
  
Bottom line: Adaptive Capacity is a safety net, not an excuse for bad data modeling.
Sustained hot partitions will still throttle; design your PK to distribute load.
```

### DynamoDB Streams

```
Streams architecture:
  Each table shard has a corresponding Stream shard
  Stream shard captures: INSERT, MODIFY, REMOVE events
  
  Event contains (depending on StreamViewType):
    KEYS_ONLY: PK/SK only
    NEW_IMAGE: new item state
    OLD_IMAGE: previous item state
    NEW_AND_OLD_IMAGES: both
  
  Retention: 24 hours
  Read: GetShardIterator → GetRecords (up to 2MB per call)
  
Lambda ESM for DynamoDB Streams:
  - Lambda polls stream shards (similar to SQS but Kinesis-like)
  - One shard iterator per Lambda poller
  - Records are ORDERED within a shard (shard = one partition's events)
  - Parallelization factor: 1–10 (concurrent batch processing within shard)
  - Failures: iterator stops at failed record (ordered delivery guarantee)
    → BisectBatchOnFunctionError: Lambda binary-searches for poison pill
```

---

## 17.4 CloudFront Deep Internals

(See Section 8 for full architecture — summarizing key deep points here)

### Cache Hit Ratio Optimization

```
Cache hit ratio = requests served from edge / total requests

Maximize by:
1. Separate cacheable from non-cacheable paths
   Cache behavior: /api/* → no cache
   Cache behavior: /static/* → 1 year TTL
   
2. Normalize cache key (remove unnecessary dimensions):
   - Don't include User-Agent in cache key (1000x fragmentation)
   - Include Accept-Encoding: gzip → CloudFront serves compressed (savings)
   - Include only necessary query params

3. Use cache control headers:
   - s-maxage overrides max-age for CloudFront specifically
   - Cache-Control: public, s-maxage=86400

4. Collapse requests (Request Collapsing / Origin Shield):
   - Cache MISS from many edge locations → many origin requests for same object
   - Origin Shield: additional regional cache layer → only 1 request to origin
   - Dramatically reduces origin load during low-cache-hit scenarios
```

### Cache Invalidation vs TTL

```
Performance tradeoff:

Short TTL (60s):
  - Stale content risk: low
  - Origin requests: high (good for dynamic content)
  - Cost: higher (more origin fetches + data transfer)
  
Long TTL (1 year):
  - Stale content risk: high (without versioned filenames)
  - Origin requests: minimal
  - Cost: minimal (served from edge)
  
Best practice for static assets:
  - Content-addressable filenames: app.a3f1c9.js (hash in filename)
  - TTL = 1 year (or max-age=31536000, immutable)
  - Deploy = new filename → cache miss for new file (old file harmlessly cached)
  - HTML: short TTL (60s) so users get new index.html pointing to new hashed assets
```

---

## 17.5 Step Functions Internal Execution

(See Section 7 for full architecture — key deep points)

### waitForTaskToken Pattern Internals

```
Step Functions generates a cryptographically unique token per Task execution.
Token is a base64-encoded, signed, opaque blob containing:
  - Execution ARN
  - State name
  - Task run ID
  - Expiry (linked to execution timeout)

The token is passed to your Lambda/ECS task.
Your code calls:
  sfn.sendTaskSuccess({ taskToken, output: JSON.stringify(result) })
  
Internally:
  - Step Functions receives SendTaskSuccess
  - Verifies token signature
  - Resumes execution at the Task state that generated the token
  - Token is invalidated after use (replay-safe)
  
Use cases:
  - Async payment gateway callbacks
  - Human approval workflows (send email with approval link that calls sendTaskSuccess)
  - External API async operations
```

### Express vs Standard Execution Internals

```
Standard:
  - All state transitions persisted to Step Functions durable store
  - Store = replicated across AZs, similar to DynamoDB
  - Execution can survive regional event and continue
  - History queryable via GetExecutionHistory API
  - Cost: $0.025/1,000 state transitions

Express:
  - State in ephemeral memory of execution engine
  - Logged asynchronously to CloudWatch Logs
  - NOT queryable via GetExecutionHistory (use CW Logs Insights)
  - Lower latency (no synchronous state persistence)
  - Cost: $1/million executions + duration
  - If execution engine crashes mid-execution → execution LOST (no recovery)
  
Choosing:
  Standard: whenever you need auditability, long-running, recoverable workflows
  Express: high-throughput, short-duration, can-retry-entire-execution scenarios
```

---

## 17.6 ECS vs Fargate vs EC2: Real Differences

### Scheduler Internals

```
ECS EC2 Launch Type:
  Control plane: ECS Scheduler
  Data plane: YOUR EC2 instances running ECS Agent

  Placement flow:
  1. ECS Scheduler receives task placement request
  2. Queries ECS Agent heartbeats for available instances
  3. Applies placement constraints + strategies
  4. Selects instance → sends task definition to ECS Agent
  5. Agent: docker pull → docker run → report status

ECS Fargate Launch Type:
  Control plane: ECS Scheduler (same API, same task definitions)
  Data plane: AWS-managed Fargate compute fleet

  Placement flow:
  1. ECS Scheduler receives task placement request
  2. Scheduler talks to Fargate scheduler (internal AWS system)
  3. Fargate allocates Firecracker MicroVM sized to task CPU/memory
  4. Container image pulled (from ECR preferred)
  5. Containers started in MicroVM
  6. ENI provisioned in your subnet (per-task VPC attachment)

EC2 (plain, no ECS):
  You manage everything: instance launch, placement, health check, scaling, AMI updates
```

### Container Runtime Differences

```
ECS EC2:
  Runtime: Docker Engine → containerd → runc
  Containers share host kernel (Linux namespaces + cgroups for isolation)
  Container escape risk: if exploit breaks out of namespace, can see host
  Kernel: same across all containers on that host
  
Fargate:
  Runtime: Docker → containerd → runc INSIDE Firecracker MicroVM
  Each task has its own kernel (in the MicroVM)
  Container escape: even if escaped, you're in a dedicated MicroVM kernel
  No other tenants visible in the MicroVM
  Stronger security posture than EC2 multi-tenant containers

Lambda:
  Runtime: Firecracker MicroVM, but no Docker layer
  Custom runtime: AWS-managed (no containerd)
  Maximum isolation: dedicated MicroVM per execution
```

### Who Manages Compute

| | You manage | AWS manages |
|-|-----------|------------|
| ECS + EC2 | Instances (AMI, patching, scaling, capacity) | ECS control plane |
| ECS + Fargate | Nothing | Compute fleet, MicroVMs, OS |
| Lambda | Nothing | Everything |

### Cost Comparison (real example)

```
Workload: 1 vCPU, 2GB RAM, 24/7 running service

EC2 (m5.large, 2vCPU 8GB, ~50% utilization):
  $0.096/hour = ~$69/month for the instance
  (fractional workload, sharing with other services = efficient)

Fargate (1 vCPU, 2GB):
  vCPU: $0.04048/vCPU-hour × 1 × 720h = $29.15
  Memory: $0.004445/GB-hour × 2 × 720h = $6.40
  Total: ~$35.55/month
  (Fargate cheaper than EC2 for this isolated workload)

Fargate Spot: ~$10.67/month (70% discount, can be reclaimed 2-min notice)

Lambda (1M invocations/month, avg 500ms each, 512MB):
  GB-seconds: (512/1024) × 0.5 × 1,000,000 = 250,000 GB-s
  250,000 × $0.0000166667 = $4.17/month
  + $0.20 request cost
  Total: ~$4.37/month
  (Lambda dramatically cheaper for bursty, short-lived workloads)
```

---

## 17.7 IAM Deep Dive

### IAM Evaluation Logic (The Complete Flow)

```
When AWS evaluates a request:

1. Authentication: Is this a valid IAM identity? (signature valid?)
2. Policy evaluation (for authorization):

EVALUATION ORDER:
  ┌─────────────────────────────────────────────────┐
  │ 1. EXPLICIT DENY (any policy)   → DENY (final) │
  │ 2. SCP (Service Control Policy) → deny if deny │
  │ 3. Resource-based policy        → allow?        │
  │ 4. Identity-based policy        → allow?        │
  │ 5. Permission boundary          → allow?        │
  │ 6. Session policy               → allow?        │
  │ Default: IMPLICIT DENY                          │
  └─────────────────────────────────────────────────┘

Cross-account access:
  Principal A (Account 1) → Resource R (Account 2)
  BOTH must allow:
    - Account 2 resource policy must allow Account 1 principal
    - Account 1 identity policy must allow action on Account 2 resource
  Either missing → DENY
```

### IAM Role Assumption (AssumeRole Internals)

```
1. Principal calls sts:AssumeRole
2. STS verifies:
   - Principal's identity policy allows sts:AssumeRole for target role ARN
   - Target role's trust policy allows this principal to assume it
3. STS generates temporary credentials:
   - AccessKeyId (ASIA... prefix)
   - SecretAccessKey
   - SessionToken (contains principal identity, role ARN, expiry, session policies)
   - Expiry: 15 minutes–12 hours
4. Credentials returned to caller
5. Caller makes API calls with temporary credentials
6. STS decodes session token on each request (no database lookup — token is self-contained signed JWT-like structure)

Lambda execution role:
  - Lambda service assumes the execution role on your behalf
  - Credentials rotated every ~6 hours (before expiry)
  - Injected into Lambda environment: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN
  - IMDS (Instance Metadata Service) proxy in Lambda serves credentials to SDK
```

### Permission Boundary vs SCP vs Resource Policy

```
SCP (Service Control Policy):
  - Applied at AWS Organization OU/account level
  - Limits MAXIMUM permissions (filter layer)
  - Even Admin user can't exceed SCP
  - Does NOT grant permissions

Permission Boundary:
  - Applied per IAM role/user
  - Limits MAXIMUM permissions for that identity
  - Does NOT grant permissions
  - Used to delegate admin rights safely

Resource-based Policy:
  - Attached to the resource (S3 bucket policy, Lambda resource policy, etc.)
  - Grants access FROM specific principals
  - Required for cross-account access

Identity-based Policy:
  - Attached to IAM user/role/group
  - Grants FROM identity TO resources/actions

Effective permissions = Intersection of all applicable policies (except for same-account resource policies which can grant standalone access)
```

### IAM in Lambda — Common Patterns

```javascript
// Least-privilege Lambda execution role example:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Orders",
      "Condition": {
        "StringEquals": {
          "dynamodb:LeadingKeys": "${aws:PrincipalTag/departmentId}"
          // Row-level access control using attribute-based access control (ABAC)
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter"],
      "Resource": "arn:aws:ssm:us-east-1:123456789:parameter/prod/orders/*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789:log-group:/aws/lambda/order-service:*"
    }
  ]
}

// Trust policy (who can assume this role):
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

### IAM Common Pitfalls

1. **Wildcard actions:** `"Action": "s3:*"` — grants ListAllMyBuckets, DeleteBucket, etc. Use explicit action list.
2. **Missing condition keys:** `s3:GetObject` on `*` without Condition → access to ALL s3 buckets in all accounts (cross-account if public bucket).
3. **Resource-based policy gotcha:** Lambda resource policy auto-added when you configure trigger (API GW, S3, etc.) — unreviewed policies accumulate over time.
4. **PassRole permissions:** Lambda that creates other Lambda functions needs `iam:PassRole` for the new function's execution role. Often forgotten.
5. **OrganizationId condition:** `aws:PrincipalOrgID` condition on S3 bucket policy allows any account in org to access — intended for cross-account org access, but also allows newly joined accounts.
6. **NotAction trap:** `"NotAction": "s3:DeleteObject"` means "allow everything EXCEPT delete" — effectively grants admin to all other services on that resource. Use explicit Allow, never broad NotAction.

---

# SUMMARY: DECISION MATRIX

## Service Selection Quick Reference

```
Use Case → Service Decision

Async event processing:
  Ordered, replay needed → Kinesis Data Streams
  Unordered, simple → SQS Standard
  Exactly-once, ordered → SQS FIFO
  Fan-out to multiple consumers → SNS → SQS

Compute:
  Sub-15min, event-driven, bursty → Lambda
  Containerized, serverless → Fargate
  Containerized, cost-efficient at scale → ECS + EC2
  Long-running, stateful, custom OS → EC2
  
Databases:
  Known access patterns, high scale, key-value → DynamoDB
  Relational, complex queries → Aurora/RDS
  Cache (sub-ms) → ElastiCache (Redis/Memcached)
  Analytics → Redshift / S3 + Athena
  
API Layer:
  Serverless, auth, throttling → API Gateway
  Container/EC2, L7 routing, gRPC → ALB
  TCP/UDP, static IP, ultra-low latency → NLB
  
Orchestration:
  Multi-step, durable, complex logic → Step Functions Standard
  High-throughput, short-duration → Step Functions Express
  Simple 2-step → SQS + Lambda
  
CDN/Edge:
  Static assets, global delivery → CloudFront
  Per-request logic at edge → Lambda@Edge / CloudFront Functions
  
Observability:
  Metrics + Alarms → CloudWatch
  Distributed tracing → X-Ray (or ADOT)
  Log analysis → CloudWatch Logs Insights
  Unified observability → CloudWatch + X-Ray + Service Lens
```

---

*Document covers: Lambda, API Gateway, S3, DynamoDB, SQS, SNS, Step Functions, CloudFront, ALB, NLB, VPC, ECS, Fargate, EC2, CloudWatch, X-Ray, IAM — with special deep dives on Lambda+SQS ESM, API Gateway→Lambda flow, DynamoDB partitioning, CloudFront internals, Step Functions execution, ECS/Fargate/EC2 differences, and IAM evaluation logic.*
