
### How ALB Works

ALB operates at the application layer (Layer 7) of the OSI model. 

Instead of just looking at IP addresses and ports (Layer 4), an ALB inspects the actual content of HTTP/HTTPS requests.

Using advanced routing rules, it can distribute traffic based on factors like:

- The request's URL path (`/orders`, `/users`)

-  The host header (e.g., `api.example.com`, `admin.example.com`)

-  HTTP headers and methods

-  Query string parameters

This intelligence allows you to direct traffic from a single ALB endpoint to different backend services, known as target groups.

### Target Groups: The Backend of Your Application

Target groups are logical groups of compute resources that receive traffic from the ALB.

- **EC2 instances**.

-  **Amazon ECS tasks** (for containerized apps).

-   **Lambda functions** (enabling serverless architectures).

-  **IP addresses**, which allows routing to on-premises resources over a VPN or Direct Connect connection.

-  Other Application Load Balancers or Network Load Balancers

### Key Features at a Glance

- **Content-Based Routing (Advanced Request Routing)**: Routes requests to different target groups based on path patterns, host headers, query strings, or HTTP methods.

- **Container & Microservices Integration**: Natively integrates with Amazon ECS and EKS, dynamically mapping ports and routing traffic to containerized applications.

- **Built-in User Authentication**: Simplifies adding user authentication by integrating with Amazon Cognito or any other OIDC-compliant identity provider.

- **Protocol Support**: Natively supports HTTP, HTTPS, WebSocket, and HTTP/2-based protocols like gRPC to enable modern application features.

- **Security Features**: Integrated with AWS WAF to protect against common web exploits and supports SSL/TLS termination at the load balancer to offload encryption overhead from backend servers.

- **Health Checks**: Monitors the health of registered targets and automatically routes traffic only to healthy ones.

- **Monitoring & Logging**: Publishes extensive real-time metrics (like request counts and latency) to CloudWatch, and can be configured to log all requests (access logs) to an S3 bucket.


### ALB vs. Other AWS Load Balancers

**Network Load Balancer (NLB)**: A Layer 4 (TCP/UDP) balancer designed for extreme performance, ultra-low latency, and handling millions of requests per second

**Gateway Load Balancer (GLB)**: Operates at Layer 3/4 and is designed to deploy, scale, and manage virtual appliances (like firewalls and intrusion detection systems)

**Classic Load Balancer (CLB)**: The legacy option that provides basic Layer 4 and Layer 7 load balancing. It's a simpler option but lacks the advanced routing features of ALB


 

