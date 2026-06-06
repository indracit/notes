## 1. What is AWS ECS?

Amazon Elastic Container Service (ECS) is a **fully managed container orchestration service** provided by AWS. It allows you to run, stop, and manage Docker containers on a cluster of virtual machines (EC2 instances) or on a serverless compute engine (AWS Fargate).

In essence, ECS solves the problem of “how do I reliably run many containers across many machines, handle failures, scale them, and connect them to other AWS services?” Without an orchestrator like ECS, you would manually manage host placement, port conflicts, health checks, and networking — which becomes impractical at scale.


## 2. Core Components & Concepts

### 2.1 Task Definition

The **task definition** is a JSON (or YAML) blueprint that describes one or more containers that should run together as a logical application.

Key fields:

- **Image**: Docker image URI (from ECR, Docker Hub, etc.)
    
- **CPU & Memory**: Resource limits (in vCPU units and MiB)
    
- **Port mappings**: Host-to-container port mapping
    
- **Environment variables**: Static or from SSM Parameter Store/Secrets Manager
    
- **Logging**: Usually `awslogs` driver to send logs to CloudWatch
    
- **Volumes**: Bind mounts or EFS filesystem attachments
    
- **IAM role** (Task Role): Permissions the container needs (e.g., access S3, DynamoDB)
    
- **Health check** command: Custom logic to determine container health
    

A task definition can have **multiple containers** (e.g., app + sidecar log shipper). They are co-located on the same host and share network namespace.

### 2.2 Cluster

An ECS **cluster** is a logical grouping of compute capacity. Clusters are regional (but can span multiple Availability Zones). Capacity can be:

- **EC2 instances** (managed by you or Auto Scaling groups)
    
- **Fargate** (serverless – no EC2 to manage)
    
- **External instances** (on-premises via ECS Anywhere)
    

Clusters are isolated from each other by default, but tasks can communicate across clusters via load balancers or service discovery.


### 2.3 Container Instance (EC2 Launch Type)


If you use the **EC2 launch type**, each EC2 instance in the cluster runs an **ECS container agent** (a binary that registers the instance to a cluster, starts/stops tasks, reports metrics). The agent communicates with the ECS control plane via API calls.

- **Instance role**: IAM role that the agent assumes (allows the instance to pull images, send logs, etc.)
    
- **User data**: Usually includes `echo ECS_CLUSTER=my-cluster >> /etc/ecs/ecs.config`
    
- **Instance metadata**: The instance must have a compatible AMI (Amazon ECS-optimized AMI) or the agent installed.

### 2.4 Task vs. Service

- **Task**: A running instance of a task definition. It is ephemeral; when it stops, it’s gone unless restarted.
    
- **Service**: A controller that maintains a desired number of identical tasks. If a task dies, the service launches a new one. Services also integrate with load balancers, auto scaling, and rolling updates.

### 2.5 Launch Types

| Feature        | EC2 Launch Type                                                                                         | Fargate Launch Type                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| Responsibility | You manage EC2 instances (patching, scaling, etc.)                                                      | AWS manages the underlying hosts                                         |
| Billing        | EC2 instance hours (plus EBS, data transfer)                                                            | Per vCPU and memory per second (minimum 1 minute)                        |
| Flexibility    | You can use spot instances, GPU instances, custom AMIs                                                  | Simpler, no host-level access                                            |
| Isolation      | Harder to achieve strong isolation between tasks (they share kernel)                                    | Each task has its own microVM (Firecracker)                              |
| Use case       | Large predictable workloads, need for cost optimisation (spot), stateful workloads with host affinities | Microservices, batch jobs, variable workloads, teams wanting minimal ops |
## 3. How ECS Works Internally (High-Level)

1. **Control plane** (managed by AWS, not visible to you) receives API calls (e.g., `RunTask`, `CreateService`). It handles scheduling, state reconciliation, and health monitoring.
    
2. **Data plane** consists of compute resources (EC2 or Fargate). The ECS agent on each instance polls the ECS API for “work” assigned to that instance.
    
3. **Scheduling**: When you run a task, ECS decides which instance (or Fargate host) meets the resource requirements (CPU, memory, port availability, placement constraints). For services, ECS uses a **scheduler** that continuously compares desired vs. actual tasks and issues `RunTask` / `StopTask` commands.
    
4. **Placement strategies** (EC2 launch type only):
    
    - **Binpack**: Minimise number of instances (pack tasks tightly)
        
    - **Spread**: Distribute tasks across AZs or instances evenly
        
    - **Random**: No preference
        
    - **Constraints** like `memberOf` (instance attribute expressions) or `distinctInstance`

## 4. Networking in ECS

### 4.1 Network Modes

The task definition’s `networkMode` determines how networking works:

- **`bridge`** (default for EC2): Containers get an internal Docker bridge IP; port mappings map container port to a dynamic host port. Suitable when multiple tasks on the same host must bind to the same container port (e.g., all tasks listen on port 80, but each gets different host port like 32768, 32769).
    
- **`host`**: Container uses the host’s network stack (no port mapping). Two tasks on same host cannot share the same host port. Only for specialised cases.
    
- **`awsvpc`** (recommended): Each task gets its own Elastic Network Interface (ENI) with a private IP from a VPC subnet. This makes each task behave like a fully isolated “micro-VM” in terms of networking. It’s mandatory for Fargate and recommended for EC2 when using service discovery or load balancing.
    
- **`none`**: No external networking.

### 4.2 Service Discovery (AWS Cloud Map)

- When tasks come and go (auto scaling, deployments), you need a way for other services to find them.
    
- ECS integrates with **AWS Cloud Map** to automatically register/de-register task IPs and ports in a DNS namespace (e.g., `myapp.internal`).
    
- Example: Service `orders` can resolve `orders.prod.internal` to the IP(s) of healthy `orders` tasks.

### 4.3 Load Balancing

- Application Load Balancer (ALB) for HTTP/HTTPS traffic – can route based on path, host.
    
- Network Load Balancer (NLB) for TCP/UDP traffic – ultra-low latency.
    
- Classic Load Balancer (legacy, not recommended for new workloads).
    
- **Integration**: ECS services automatically register and deregister tasks with the target group. Health checks are passed from the load balancer to ECS (task health checks).

### 4.4 Service Discovery vs. Load Balancing

|Feature|Service Discovery|Load Balancer|
|---|---|---|
|Traffic distribution|DNS round-robin or client-side|Intelligent (least outstanding requests, etc.)|
|Health check|DNS entry can return multiple IPs; unhealthy tasks are removed from DNS|Advanced health checks, draining|
|Use case|Internal gRPC/Thrift, stateful services, sticky sessions not needed|Web frontends, microservices needing rich routing|

## 5. Security & IAM

### 5.1 IAM Roles in ECS (Three distinct roles)

1. **Instance role** (EC2 launch type): Assigned to EC2 instance. Allows the ECS agent to call ECS APIs, pull images from ECR, send logs to CloudWatch.
    
2. **Task role**: Assumed by the containers themselves. Grants permissions to AWS services (e.g., `s3:GetObject`, `dynamodb:PutItem`). Uses ECS’s credential injection — a secure endpoint accessible only to that task.
    
3. **Execution role** (Fargate & EC2): Used by the ECS agent to pull images from private registries (ECR, Docker Hub with authentication), write logs, and decrypt secrets.

### 5.2 Security Groups

- In **awsvpc** mode, each task has its own ENI, so you can assign a security group directly to the task.
    
- In bridge/host mode, you assign security groups to the EC2 instance – less granular.

### 5.3 Secrets Management

- Store secrets in **AWS Secrets Manager** or **Parameter Store (SecureString)**.
    
- In task definition, reference them as `valueFrom: arn:aws:ssm:...`. ECS injects them as environment variables or as files (with `secrets` block for Fargate or `dockerSecrets` for EC2).

## 6. Auto Scaling

### 6.1 Service Auto Scaling

- ECS services can automatically adjust the desired task count based on CloudWatch metrics.
    
- Supported scaling targets:
    
    - **Target tracking** (e.g., keep average CPU at 50%)
        
    - **Step scaling** (custom CloudWatch alarm with steps)
        
    - **Scheduled scaling** (predictable patterns)
        
- Application Auto Scaling is the underlying service.
    

### 6.2 Cluster Auto Scaling (EC2 launch type)

- As tasks are placed onto EC2 instances, the cluster may run out of capacity.
    
- **Cluster Auto Scaling** (or Managed Scaling) adds/removes EC2 instances to the Auto Scaling Group based on “capacity providers” that track memory and CPU reservation.
    
- **Capacity providers** decouple capacity management: you define a strategy (e.g., “use 30% on-demand, 70% spot, fallback to Fargate if needed”).

### 6.3 Fargate Auto Scaling

- No cluster scaling needed – the service scaling simply requests more tasks, and Fargate provisions the underlying compute in seconds.

## 7. Deployment Strategies (Services)

- **Rolling update** (default): ECS stops old tasks one by one, starts new ones, controlled by `maximumPercent` and `minimumHealthyPercent`.
    
- **Blue/Green deployment** via AWS CodeDeploy: Two independent task sets (blue = current, green = new). Traffic is shifted using a load balancer listener rule. Allows automated testing, instant rollback.
    
- **External deployments**: You can use tools like Spinnaker, Jenkins, or AWS Copilot to implement custom strategies (e.g., canary, A/B testing).
  
## 8. Statefulness & Persistent Storage

By default, containers are ephemeral. For state, ECS provides:

- **Bind mounts**: Mount a host directory into container (EC2 launch type only). Not portable if instance is replaced.
    
- **Docker volumes**: Using an EBS volume provisioned by Docker (rarely used directly with ECS).
    
- **Amazon EFS** (Elastic File System): Network file system that can be mounted by any task in any AZ. Highly recommended for stateful workloads (e.g., content repositories, shared config). Fargate natively supports EFS.
    
- **FSx for Lustre** (high-performance compute).
    
- **External storage**: Third-party drivers (e.g., Portworx) for replicated block storage.


## 9. Observability & Logging

- **CloudWatch Logs**: `awslogs` driver sends stdout/stderr to log groups. Can attach to task definition.
    
- **Container insights**: Provides metrics (CPU, memory, network) per task or per cluster. Two tiers:
    
    - **Basic**: aggregate cluster metrics (free)
        
    - **Enhanced**: per-task metrics, service-level aggregations (paid)
        
- **AWS X-Ray**: Tracing distributed requests. The X-Ray daemon can run as a sidecar container in the task definition.
    
- **FireLens** (for advanced logging): Uses Fluent Bit as a sidecar to route logs to many destinations (S3, Kinesis, Elasticsearch, Datadog, etc.) without changing application code.

## 11. Typical Workflow to Run an Application on ECS

1. **Containerize** app (Dockerfile), build image, push to Amazon ECR (or any registry).
    
2. **Define task definition** (JSON/Terraform/CloudFormation). Specify image, CPU, memory, environment, ports, IAM role, logging.
    
3. **Create cluster** (Fargate or EC2). For EC2, launch instances with the ECS-optimized AMI.
    
4. **Create a service** referencing the task definition, desired count, load balancer (optional), network configuration.
    
5. **Service** runs tasks. ECS monitors health. If a task fails, it restarts. If scaling is configured, adjusts count.
    
6. **Update** by registering new task definition revision and updating service. Rolling deployment occurs automatically.

## 12. Common Pitfalls & Theoretical Constraints

- **Task networking in bridge mode**: Random host ports make service discovery hard; prefer `awsvpc`.
    
- **Storage in Fargate**: Only ephemeral storage (up to 200 GB) or EFS. No EBS or host bind mounts.
    
- **Task size limits**: Fargate supports up to 16 vCPU, 120 GB memory; EC2 can go higher with larger instances.
    
- **IAM permissions**: Tasks don’t inherit the instance role; you must assign a task role.
    
- **Placement constraints**: Not available on Fargate (AWS chooses the host).
    
- **Daemon services**: No native “run exactly one per instance” like DaemonSet in Kubernetes; you must use `placementConstraints` on EC2 with `distinctInstance` and service count equal to number of instances, but it’s manual.

## 13. Summary Diagram of ECS Architecture (conceptual)


```text

[User/CI/CD] → [ECS API]
                  ↓
             Control Plane
        (State reconciliation,
         scheduling, scaling)
                  ↓
        +--------+--------+ (Capacity Providers)
        |                 |
   EC2 Launch           Fargate
   (Cluster of EC2)   (Serverless)
        |                 |
   ECS Agent               |
        |                 |
   [Task A] [Task B]   [Task A] [Task B]
        |                 |
   (ENI bridge)      (AWVPC ENI per task)
        |                 |
   Load Balancer ←→ Service Discovery
                  ↓
              (VPC/Subnets)
```



## Conclusion

Amazon ECS provides a robust, scalable, and deeply integrated container orchestration service on AWS. It abstracts away low-level decisions (like which host to place a task on) while giving you fine-grained controls when needed (placement strategies, network modes, capacity providers). With Fargate, it becomes a fully serverless compute layer, reducing management overhead to nearly zero. Understanding its core concepts — task definitions, services, clusters, networking modes, and IAM roles — is essential to building reliable containerised applications on AWS.