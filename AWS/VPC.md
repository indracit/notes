
### What is AWS VPC?

At its simplest, an **Amazon VPC** is a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. It is often described as creating your own "data center" inside AWS, with complete control over your network environment.

You have full control over:

- **IP Address Range:** Your private CIDR block (e.g., `10.0.0.0/16`).
-  **Subnets:** Dividing your IP range into smaller segments.
- **Route Tables:** Controlling traffic flow between subnets and the internet.
- **Gateways & Security:** Managing internet access, VPN connections, and firewalls.

## Core Components of a VPC (The Building Blocks)

### 1. CIDR Block (IP Addressing)

When you create a VPC, you must assign a primary IPv4 CIDR block. The allowed range is from `/16` (65,536 IPs) to `/28` (16 IPs). AWS reserves the first 4 IP addresses and the last 1 IP address in each subnet for networking purposes.

- **Example:** `10.0.0.0/16` (This creates 65,536 IP addresses from `10.0.0.0` to `10.0.255.255`).
-  **Best Practice:** Use private IP ranges from RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, or `192.168.0.0/16`. Avoid overlapping CIDRs if you plan to connect VPCs.

### 2. Subnets

A subnet is a smaller range of IP addresses _within your VPC_, deployed to a single **Availability Zone (AZ)**. Subnets are AZ-specific; they cannot span AZs.

- **Public Subnet:** Has a direct route to an Internet Gateway. Resources here (like web servers) can have public IPs and be reached from the internet.
    
- **Private Subnet:** Does _not_ have a direct route to an Internet Gateway. Resources here (like databases) are isolated from the internet.
    
- **VPN-Only Subnet:** Has a route to a Virtual Private Gateway (for corporate network access) but not to an IGW.

### 3. Route Tables

A route table contains a set of rules, called _routes_, that determine where network traffic is directed. Every subnet must be associated with a route table.

- **Main Route Table:** Created automatically with the VPC. Controls all subnets not explicitly associated with another table.
- **Custom Route Table:** You create these for specific subnets.
- **Local Route:** Every route table has a default "local" route (e.g., `10.0.0.0/16 → local`), allowing all resources in the VPC to communicate privately.

**Example Route in a Public Subnet Table:**

- Destination: `0.0.0.0/0` (any internet traffic)
- Target: `igw-xxxxxxxx` (Internet Gateway ID)

### 4. Internet Gateway (IGW)

A horizontally-scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet. It serves two purposes:

- Provide a target for internet-bound traffic in route tables.
- Perform NAT (Network Address Translation) for instances with public IPs.

### 5. NAT Gateway / NAT Instance

Allows instances in a _private_ subnet to initiate outbound traffic to the internet (e.g., for software updates) while preventing the internet from initiating connections to them.

- **NAT Gateway (Managed):** AWS managed, highly available (within one AZ), scales automatically. Preferred for production.
    
- **NAT Instance (Self-managed):** An EC2 AMI you configure. Older, less scalable, but cheaper for testing.
    

**How it works:** A private subnet instance sends traffic to the NAT Gateway (which sits in a public subnet). The NAT Gateway forwards it to the IGW, receives the response, and sends it back.


### 6. Virtual Private Gateway (VPG)

The component on the AWS side of a VPN connection. It allows you to connect your VPC to your on-premises network via an encrypted VPN tunnel or AWS Direct Connect.

### 7. VPC Peering

A networking connection between two VPCs (same or different AWS account, same or different region) that routes traffic between them using private IPv4/IPv6 addresses.

- **Rules:** No transitive peering (if VPC A peers with B and C, B and C cannot communicate via A). You need direct peering or a Transit Gateway.

### 8. AWS Transit Gateway

A central hub that connects multiple VPCs and on-premises networks. Solves the transitive peering problem. Acts as a cloud router.

### 9. VPC Endpoints (Interface & Gateway)

Allow you to privately connect your VPC to supported AWS services (like S3, DynamoDB, CloudWatch) _without_ requiring an IGW, NAT, or VPN.

- **Gateway Endpoint (S3 & DynamoDB only):** Free. Adds a route in your route table pointing to the service.
    
- **Interface Endpoint (Powered by AWS PrivateLink):** Creates an Elastic Network Interface (ENI) in your subnet with a private IP. Use for services like ECR, CloudFormation, or your own services.

## Security in a VPC

### Security Groups (SG) - _Stateful Firewall_

- Acts as a _virtual firewall for an ENI (EC2 instance)_.
    
- **Stateful:** If you allow inbound traffic, the outbound return traffic is automatically allowed, regardless of outbound rules.
    
- Supports "allow" rules only (no explicit deny).
    
- Can reference other SGs by ID as rules (e.g., "Allow web server SG from app server SG").
    
- Evaluates **all rules** before deciding to allow traffic.
    

### Network ACLs (NACL) - _Stateless Firewall_

- Acts as a _firewall for a subnet_ (applies to all instances in that subnet).
    
- **Stateless:** Inbound and outbound rules are evaluated separately. Return traffic must be explicitly allowed.
    
- Supports both "allow" and "deny" rules.
    
- Rules are evaluated in order (lowest to highest rule number).
    
- Great for adding a simple layer of defense (e.g., deny a specific IP address).