
## AWS IAM (Identity and Access Management)


AWS IAM is the fundamental security service that controls who can access what in your AWS account.

It’s a **global** (region‑agnostic) service, provided at no additional cost, that lets you manage identities, permissions, and authentication.

### Core Purpose of IAM

- **Authentication** – verify the identity of a principal (user, application, or service).
- **Authorization** – determine which actions a principal is allowed or denied to perform on which AWS resources.

### Key Components

#### 1. **IAM Users**

- An IAM user represents a **person or application** that needs long‑term access to AWS.

- Each user has:
    
    - A **friendly name** (e.g., `john.doe`)
        
    - An optional **password** for the AWS Management Console.
        
    - Up to two **access key ID + secret access key** pairs for the CLI, SDK, or API.
        
- By default, new users have **no permissions** – they must be granted access.
#### 2. **IAM Groups**

- A group is a **collection of IAM users**.
    
- Permissions are attached to the group; all members inherit them.
    
- A user can belong to multiple groups (union of permissions).
    
- Groups cannot contain other groups (no nesting).  
    _Best practice:_ Assign permissions via groups, never directly to individual users.

#### 3. **IAM Roles**

- A role is an **identity with permissions** but **no long‑term credentials** (password or access keys).

- Instead, a role is **assumed** temporarily to obtain a short‑lived (up to 12 hours, default 1 hour) set of credentials.

- Roles are used by:
	 - **AWS services** (e.g., an EC2 instance needs permissions to write to S3).
	 - **Cross‑account access** (User in Account A assumes a role in Account B).
	 - **Federated users** (e.g., corporate Active Directory or Google login).
	
- **Service roles** – a role that an AWS service assumes to act on your behalf.
- **Service‑linked roles** – a special type predefined by an AWS service (e.g., AWS Config) with permissions tightly coupled to that service.
	
#### 4. **Policies**

- Policies are **JSON documents** that define permissions (what actions are allowed/denied on which resources under what conditions).
    
- IAM uses a **policy evaluation logic** – explicit `Deny` always overrides any `Allow`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-bucket",
      "Condition": {
        "IpAddress": { "aws:SourceIp": "203.0.113.0/24" }
      }
    }
  ]
}

```

**Fields explained:**

- **Version** – policy language version (always `2012-10-17`).
- **Statement** – one or more individual permissions.
	-  **Effect** – `Allow` or `Deny`.
	- **Action** – AWS API action(s) (wildcards allowed: `s3:*`, `ec2:Describe*`).
	-  **Resource** – ARN(s) of the AWS resource (e.g., `arn:aws:s3:::my-bucket/*`).
	- **Condition** (optional) – additional constraints (IP address, time, MFA present, etc.).

**Types of policies:**

- **AWS managed policies** – created & maintained by AWS (e.g., `ReadOnlyAccess`). Cannot be changed.
    
- **Customer managed policies** – you create, you maintain versioning, you manage updates.
    
- **Inline policies** – embedded directly into a single user, group, or role (not recommended for large setups).
    
- **Permissions boundaries** – set the **maximum** permissions a role/user can be given (a “guardrail”).
    
- **Service Control Policies (SCPs)** – used with AWS Organizations to restrict what is allowed across multiple accounts (often called “allowlist/denylist” at the account level).

### IAM Evaluation Logic (Summary)

1. By default, all actions are `Deny` (implicit deny).
    
2. An explicit `Allow` overrides the default deny.
    
3. An explicit `Deny` overrides any `Allow` (evaluated last).


### Authentication Options

|Principal Type|How to authenticate|
|---|---|
|Human (Console)|Username + password (+ optional MFA)|
|Human (CLI/SDK)|Access key ID + secret access key|
|AWS Service (e.g., EC2)|IAM role (auto‑provides temporary credentials)|
|Federated user (SAML/OIDC)|Assume role via identity provider|
|Cross‑account user|Assume role from another account|

### Simple Example Walkthrough: Creating a “ReadOnlyS3” group

1. **Create group** `s3-readonly-group`.
2. **Attach AWS managed policy** `AmazonS3ReadOnlyAccess`
3. **Add user** `alice` to the group.
4.  Alice can now list and download objects from any S3 bucket in the account (but cannot upload, delete, or change bucket settings).
5. To restrict only to one bucket, create a **customer managed policy** like:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": ["arn:aws:s3:::alice-bucket", "arn:aws:s3:::alice-bucket/*"]
  }]
}
```

