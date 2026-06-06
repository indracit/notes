
AWS Lambda is a **serverless, event-driven compute service** that lets you run code without provisioning or managing servers. You only pay for the compute time you consume — there is no charge when your code is not running

### Core Concepts

#### 1. Serverless ≠ No Servers

"Serverless" means AWS manages all the underlying infrastructure (OS, patching, scaling, availability). You focus solely on writing code.

#### 2. Function

A Lambda function is your code packaged with its dependencies. It can be written in Node.js, Python, Java, Go, Ruby, C#, PowerShell, or custom runtimes (via Runtime API).


#### 3. Event-Driven

Lambda functions are triggered by **events** from other AWS services (S3 upload, DynamoDB change, API Gateway request) or external sources (HTTP, queue messages).

#### 4. Stateless by Design

Each execution runs in a fresh, isolated environment. If you need persistence, use external services like DynamoDB, S3, or ElastiCache.


### How Lambda Works

1. **Upload code** – Zip or container image (up to 10 GB for images, 250 MB for zipped code).
2. **Configure trigger** – Define what event invokes the function (e.g., S3 bucket creation).
3. **Lambda creates execution environment** – When first invoked, Lambda allocates CPU, memory, network, and spins up a runtime container.
4. **Invoke** – The function runs. Subsequent requests may reuse the same container (called **warm start**), dramatically reducing latency.
5. **Scale automatically** – Lambda launches as many parallel execution environments as needed, up to region limits (default 1,000 concurrent executions, can be raised).
6. **Idle → Shutdown** – After a period of inactivity (typically 5–15 minutes), Lambda terminates unused containers.

### Invocation Types

|Type|Description|Use case|
|---|---|---|
|**Synchronous** (`RequestResponse`)|Caller waits for result.|API Gateway, CLIs, SDK calls.|
|**Asynchronous** (`Event`)|Lambda queues event and retries on error (2x backoff, up to 2×).|S3 events, SNS, CloudWatch Events.|
|**Poll‑based**|Lambda polls a stream/queue and invokes your function with batches.|DynamoDB Streams, Kinesis, SQS.|

### Common Triggers & Integration


- **HTTP endpoints** – Amazon API Gateway (REST/HTTP/WebSocket)
    
- **File uploads** – S3 bucket events (object creation/deletion)
    
- **Database changes** – DynamoDB Streams, Aurora (Data API)
    
- **Message queues** – SQS, SNS, EventBridge
    
- **Schedule** – CloudWatch Events (cron / rate expressions)
    
- **Code pipeline** – CodeCommit, CodePipeline
    
- **Custom apps** – SDK, AWS CLI, Lambda URLs (direct HTTPS endpoint)
	

### Key Features

#### Automatic Scaling

- Scales from 0 to thousands of concurrent executions in seconds.

- Each function instance handles one request at a time (by default). For stream/poll-based sources, you can control batching and concurrency.

#### Resource Model

- **Memory**: 128 MB to 10,240 MB (in 1 MB increments). CPU is proportional to memory (roughly 1.8 GB RAM per vCPU). More memory = more CPU = faster execution.
    
- **Timeout**: Max 15 minutes (900 seconds). Not suitable for long-running jobs.
    
- **Ephemeral storage**: 512 MB to 10,240 MB (default 512 MB) in `/tmp`.

#### Deployment Options

- **Zip upload** (max 250 MB after extraction) – best for small functions.
    
- **Container images** (max 10 GB) – support custom runtimes, larger dependencies, familiar tooling.

#### Security & Networking

- IAM execution role grants permissions to other AWS services.
    
- Can run inside a VPC to access private resources (RDS, ElastiCache, etc.) – requires VPC configuration with ENIs.
    
- Environment variables (encrypted with KMS).

### Versioning & Aliases

- **Versions** – Immutable snapshots of your function code & config (`$LATEST` + numbered versions).
    
- **Aliases** – Pointers to specific versions (e.g., `PROD`→v5, `DEV`→v3). Allows traffic shifting (canary deployments).


### Lambda Layers

Reusable components (libraries, custom runtimes, configuration) shared across multiple functions. Each function can use up to 5 layers.


### Use Cases

|Use case|Why Lambda fits|
|---|---|
|**REST / GraphQL backends**|API Gateway + Lambda = no server to scale.|
|**File processing** (resize, watermark)|Trigger on S3 upload, process in parallel.|
|**Stream processing** (real‑time analytics)|Process Kinesis or DynamoDB streams in micro‑batches.|
|**Scheduled tasks** (cron jobs)|Replace EC2 crontab with CloudWatch Events.|
|**Chatbots / webhooks**|Short‑lived, unpredictable traffic.|
|**Infrastructure automation**|Respond to CloudTrail events, auto‑tag resources.|
|**Mobile / IoT backends**|Low latency, pay-per-use.|

### Limitations & Constraints (Important)

| Constraint               | Limit                                        |
| ------------------------ | -------------------------------------------- |
| Execution timeout        | **900 seconds (15 min)**                     |
| Temporary disk (`/tmp`)  | **512 MB – 10 GB**                           |
| Deployment package (zip) | 250 MB (unzipped)                            |
| Container image          | 10 GB                                        |
| Memory range             | 128 MB – 10,240 MB                           |
| Environment variables    | 4 KB total                                   |
| Request/response payload | **6 MB** (sync) / **256 KB** (async)         |
| Concurrency (soft limit) | **1,000** per region (default, can increase) |
| Function layers          | 5 layers per function                        |
| File descriptors         | 1,024                                        |
| Processes/threads        | 1,024                                        |

**Hard-to‑debug issues**:

- **Cold starts** – When a new container spins up (code load + runtime init), can add 100 ms – 1+ seconds. Mitigations: Provisioned Concurrency (pay for warm instances), smaller deployment size, using SnapStart (Java).
    
- **Statelessness** – Do not rely on local disk or memory between invocations.
    
- **VPC ENI latency** – Functions inside a VPC can have slower cold starts (ENI attachment).

### Best Practices

1. **Keep functions small & single‑purpose** – One task per function (e.g., `resize-image`, `send-welcome-email`).
    
2. **Use environment variables** for configuration (DB tables, stage names). Never hardcode secrets – use AWS Secrets Manager or Parameter Store.
    
3. **Minimize deployment size** – Fewer dependencies = faster cold starts. Use layers for shared libraries.
    
4. **Leverage reserved concurrency** to limit runaway functions and guarantee throughput for critical functions.
    
5. **Use dead‑letter queues (DLQ)** or **destination** for failed asynchronous invocations (send failures to SQS/SNS/EventBridge).
    
6. **Enable CloudWatch Logs** – Lambda automatically logs output (console.log / print). Structure logs with JSON for easy querying.
    
7. **Idempotency** – Especially for asynchronous or SQS triggers. An event may be delivered more than once.
    
8. **Test locally** with AWS SAM, Serverless Framework, or Lambda RIE (Runtime Interface Emulator).



### 1. Memory & CPU: The Core Performance Lever

 You set the **memory** for your function, and AWS proportionally allocates **CPU power**, network bandwidth, and other resources.

**How to Configure**:

You can set memory between 128 MB and 10,240 MB (10 GB) in 1-MB increments[](https://docs.aws.amazon.com/lambda/latest/dg/configuration-memory.html?refid=b28d8305-f5fb-4858-9ae6-04a78cfcc154). The default is 128 MB

**The Memory-CPU Link**: CPU is allocated in direct proportion to the memory configured. For example:

At **1,792 MB** of memory, your function gets the equivalent of **1 full vCPU**

At **10,240 MB** (the maximum), your function has access to up to **6 vCPUs**

**How to Choose**: Finding the "sweet spot" for memory involves balancing speed and cost. Here's a practical guide:

**Start Simple (128 MB)**: This is only recommended for very simple, non-CPU-intensive tasks, like routing events to other AWS services

**Follow the Performance (for CPU/Network-bound tasks)**: If your function is CPU, network, or memory-bound, increasing memory (and thus, CPU) can dramatically improve performance by removing a bottleneck

**Find Your "Sweet Spot" (for Cost Optimization)**: The relationship between memory and cost isn't always linear. AWS charges for compute time in **GB-seconds** (Memory in GB × Duration in seconds). So, a counter-intuitive strategy can be very effective: **Increasing memory may lower your overall cost**.

Use the open-source **AWS Lambda Power Tuning** tool. It runs your function across multiple memory configurations and generates a visual graph showing the best performance vs. cost trade-off for your specific workload

**The `ephemeralStorage` Setting**: You can configure the size of the `/tmp` directory, a temporary, non-persistent scratch space available to your function code. Setting this in an Infrastructure as Code (IaC) template like AWS SAM would look like this:

```yaml

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: my-function-code/
      Handler: app.handler
      Runtime: nodejs18.x
      MemorySize: 3008 # Memory setting in MB
      EphemeralStorage:
        Size: 5120    # Ephemeral storage size in MB (optional, example: 5 GB)
```


### 2. Ephemeral Storage (`/tmp`)

Each Lambda function instance has access to a temporary, non-persistent scratch space at the `/tmp` directory for writing files during execution. The data is deleted when the execution environment is shut down

**Configuration**: You can configure this from 512 MB (default) up to 10,240 MB (10 GB)

**Use Cases**: Use `/tmp` for:

- Downloading large files from S3 for processing.
-  Generating and manipulating reports (e.g., creating PDFs or Excel files) before uploading them elsewhere.
- Storing temporary data for a single invocation.
- **Pricing**: The first 512 MB are free. Storage configured above that is charged based on the amount and the duration your function is running

### 3. Concurrency Management

Concurrency is the number of instances of your function that are serving requests at the same time. AWS Lambda has a **regional concurrency limit** (default 1,000 per account) to protect your account from runaway processes[](https://notes.kodekloud.com/docs/AWS-Certified-Developer-Associate/Serverless/Limits-Concurrency/page). You have two primary ways to manage this:

#### **A. Reserved Concurrency**

This provides a safety and isolation mechanism by guaranteeing a maximum number of concurrent executions for a specific function. Think of it as a "cap" or a "guarantee"

- **How It Works**: When you set a reserved concurrency of, say, 200, this reserves 200 units from your account's regional concurrency pool and dedicates them to that function. It will never scale beyond that limit, which protects downstream resources (like a database) from being overwhelmed.

  **Impact on Other Functions**: It also provides isolation. If this function uses its 200 concurrent executions, other functions cannot consume them, ensuring this function always has the capacity to run
**Pricing**: Configuring reserved concurrency is **free**.

#### **B. Provisioned Concurrency**

This is a feature designed to eliminate **cold starts**, which occur when a new instance of your function needs to be initialized

**How It Works**: Provisioned concurrency instructs AWS to keep a specified number of execution environments fully initialized and ready to respond immediately

It is configured on a **version** or **alias** of your function, not the `$LATEST` version

**Perfect For**: Latency-critical applications, like interactive web or mobile backends, where even a sub-second delay is unacceptable

**Pricing**: Because you are paying for compute capacity to be "on" and waiting, provisioned concurrency **incurs additional charges**. You are billed for all provisioned capacity, regardless of whether it's serving requests.

**Pricing Strategy**: A common and effective strategy is to use **Application Auto Scaling** to schedule changes to your provisioned concurrency. For example, you can set it to a high number during peak business hours and scale it down to zero at night, significantly reducing costs

