

### 1. The AWS Account Root User (The Master Key)

When you first create an AWS account, you start as the **Root User**.
The ultimate owner. Has complete, unrestricted access to _everything_ in the account – billing, all services, every corner of the house.

### 2. IAM Users (Individual Keycards)

An **IAM User** is a person, application, or service that needs access to AWS.
 **What it is:** A unique identity _inside_ your AWS account. Think of it as an employee with a name (`john.doe`, `jenkins-server`).

**How it accesses AWS**

**Username + Password:** For logging into the AWS Management Console (the web UI)
 **Access Key ID + Secret Access Key:** For using the command line (AWS CLI) or writing code (SDKs). _Treat these like passwords!_

 **Key Principle:** **Least Privilege** – give each user _only_ the permissions they need, nothing more. John in marketing doesn't need to delete production databases.

### 3. IAM Groups (Departments)

**What it is:** A collection of IAM Users. You attach permissions to the **Group**, and every user in that group inherits them

- **Example:**
    
    - `Developers` group -> gets permissions to launch servers (EC2) and use databases (RDS).
    
    - `Auditors` group -> gets read-only permissions to view logs and billing.
        
    - `Admins` group -> gets wide, but _still less than Root User_, permissions.
### 4. IAM Policies (The Rulebook Documents)

This is the core of IAM. A **Policy** is a JSON (text) document that explicitly defines what is allowed or denied.

```json
{
  "Effect": "Allow",
  "Action": "s3:ListBucket",   // Allowed to list files in S3
  "Resource": "arn:aws:s3:::my-company-logs" // ONLY in this specific bucket
}
```

- **How they work:** You write or choose a policy, then **attach** it to a User, Group, or Role.
    
- **Types:** AWS provides hundreds of pre-built policies (`AmazonS3ReadOnlyAccess`, `EC2FullAccess`). You can also write your own custom policies.

### 5. IAM Roles (Temporary Permissions for "Things")

A **Role** is like a "costume" or "identity" that _someone or something_ can assume temporarily.

- **Crucial difference from Users:** A User has permanent credentials (a password/keys). A Role has _temporary_ credentials you get on demand.

- **Who can assume a Role?**
    
    - **An IAM User** (e.g., you "switch roles" to become an Auditor for 1 hour).
        
    - **An AWS Service** (e.g., an EC2 server needs to read a file from S3 – you give it a Role, not a user account).
        
    - **A User from another AWS account** (federated access).

- **Simple Example:** You have a server (EC2) that needs to write logs to S3 storage. You do **not** create an IAM User for that server. You:
    
    1. Create a Role called `ServerLogWriter`.
        
    2. Attach a policy allowing `s3:PutObject` (write permission).
        
    3. Attach that Role to the EC2 server. The server now _temporarily_ has permission.
