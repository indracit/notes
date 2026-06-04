

#### **1: Create Artifact Registry Repository**

Run these commands from your local machine (with `gcloud` configured):

```bash

# Set your variables
export REGION="us-central1"  # Change to your preferred region
export PROJECT_ID="your-project-id"  # Your GCP project ID
export REPO_NAME="my-app-repo"

# Create the repository
gcloud artifacts repositories create $REPO_NAME \
    --repository-format=docker \
    --location=$REGION \
    --description="Docker repository for my app"
```

#### **2: Grant VM Access to Pull Images**

SSH into your VM and run these commands:

```bash

# On your VM - authenticate Docker to pull from Artifact Registry
gcloud auth configure-docker $REGION-docker.pkg.dev

# Test the authentication (optional - verifies it works)
docker pull $REGION-docker.pkg.dev/$PROJECT_ID/$REPO_NAME/hello-world:latest

```

If `gcloud` isn't installed on your VM, install it first:

```bash

# Install Google Cloud SDK on Ubuntu/Debian VM
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates gnupg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-sdk

```


First, ensure the IAM Credentials API is enabled in your project. Your user account will need the **Service Account Admin (`roles/iam.serviceAccountAdmin`)** and **Workload Identity Pool Admin (`roles/iam.workloadIdentityPoolAdmin`)** roles.

```bash

gcloud services enable iamcredentials.googleapis.com

```


#### 4.  **Grant Required IAM Roles**

```bash

# Grant Artifact Registry Writer permission (to push Docker images)
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:1060720871947-compute@developer.gserviceaccount.com" \
    --role="roles/artifactregistry.writer"
    
# Grant Service Account User permission (allows using this account for authentication)
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:1060720871947-compute@developer.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser"
    
```

#### **3 : Create Workload Identity Pool and Provider**

```bash

# --- Set your variables (Customize these) ---
export PROJECT_ID="your-gcp-project-id"
export WIF_POOL_NAME="github-actions-pool"
export WIF_PROVIDER_NAME="github-actions-provider"
export GITHUB_REPO="indracit/cicd" # e.g., "my-org/my-app"

# --- Create Workload Identity Pool ---
gcloud iam workload-identity-pools create $WIF_POOL_NAME \
    --project="$PROJECT_ID" \
    --location="global" \
    --display-name="GitHub Actions Pool"
    
# --- Create OIDC Provider within the Pool for GitHub ---
gcloud iam workload-identity-pools providers create-oidc $WIF_PROVIDER_NAME \
    --project="$PROJECT_ID" \
    --location="global" \
    --workload-identity-pool="$WIF_POOL_NAME" \
    --display-name="GitHub Actions Provider" \
    --attribute-mapping="google.subject=assertion.sub" \
    --attribute-condition="assertion.repository_owner == 'indracit'" \
    --issuer-uri="https://token.actions.githubusercontent.com"

```

#### **4: Grant the Service Account Access to the Pool**


Now, allow the newly created identity pool to impersonate your existing GCP service account (the one that has permissions to push to Artifact Registry and, if needed, manage Compute Engine instances).


```bash

# --- Set your Service Account email ---
export SERVICE_ACCOUNT_EMAIL="your-service-account@$PROJECT_ID.iam.gserviceaccount.com"

# --- Get the Workload Identity Pool ID ---
export WIF_POOL_ID=$(gcloud iam workload-identity-pools describe $WIF_POOL_NAME \
    --project="$PROJECT_ID" \
    --location="global" \
    --format="value(name)")
    
 # --- Bind the Service Account to the Identity Pool ---
gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --project="$PROJECT_ID" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/$WIF_POOL_ID/attribute.repository_owner/$GITHUB_REPO"   
    
    
# check service account
gcloud iam service-accounts list --project="$PROJECT_ID"

   
```


## **Grant the `roles/iam.serviceAccountTokenCreator` role**

### **Find your Workload Identity Pool principal**

```bash

gcloud iam workload-identity-pools describe github-actions-pool \
    --project=$PROJECT_ID \
    --location="global" \
    --format="value(name)"
    
    
```

### **Grant the `roles/iam.serviceAccountTokenCreator` role**

```bash

gcloud iam service-accounts add-iam-policy-binding $SERVICE_ACCOUNT_EMAIL \
    --project=$PROJECT_ID \
    --role="roles/iam.serviceAccountTokenCreator" \
    --member="principalSet://iam.googleapis.com/1060720871947/locations/global/workloadIdentityPools/github-actions-pool/attribute.repository_owner/indracit"

```


check pushed images

```bash

gcloud artifacts docker images list us-central1-docker.pkg.dev/project-9c77ce7d-158f-4f66-926/my-app-repo --include-tags
```


### for broader access
```bash

gcloud iam service-accounts add-iam-policy-binding 1060720871947-compute@developer.gserviceaccount.com \
    --role="roles/iam.serviceAccountTokenCreator" \
    --member="principalSet://iam.googleapis.com/projects/1060720871947/locations/global/workloadIdentityPools/github-actions-pool/*"
```


## 1. **Verification Commands**

```bash

gcloud artifacts repositories list --location=us-central1

```


### **2. Check Workload Identity Pool**

```bash

gcloud iam workload-identity-pools list --location="global"

```

### **3. Check IAM bindings on the default service account**

```bash

gcloud iam service-accounts get-iam-policy 1060720871947-compute@developer.gserviceaccount.com

```

### **4. Check VM instances**

```bash

gcloud compute instances list --filter="name=cicd-vm"

```

### **5. Check Cloud SQL instances**

```bash

gcloud sql instances list --filter="name=cicd-db"

```

### **6. Check VPC peering**

```bash

gcloud services vpc-peerings list --network=default
```

### **7. Check firewall rules**

```

gcloud compute firewall-rules list --filter="name~'allow-http|allow-https'"

```

### **8. Check static IP addresses**

```bash

gcloud compute addresses list --filter="name=cicd-vm-ip"


```

# Complete Clean up 

```bash

#!/bin/bash
# ============================================================
# Complete Cleanup for CI/CD Practice Resources
# - GCE VM, Cloud SQL, VPC peering, firewall rules, static IP
# - Artifact Registry images & repository
# - Workload Identity Pool & Provider
# - IAM bindings from service account (default and/or custom)
# - Optional: custom service account deletion
# ============================================================

set -e  # stop on error (optional)

# -------------------------------
# CONFIGURATION
# -------------------------------
export PROJECT_ID=$(gcloud config get-value project)
export REGION="us-central1"
export ZONE="us-central1-a"
export INSTANCE_NAME="cicd-vm"
export SQL_INSTANCE="cicd-db"
export REPO_NAME="my-app-repo"
export WIF_POOL="github-actions-pool"
export WIF_PROVIDER="github-actions-provider"
export SERVICE_ACCOUNT_DEFAULT="${PROJECT_NUMBER:-1060720871947}-compute@developer.gserviceaccount.com"  # adjust if needed
export SERVICE_ACCOUNT_CUSTOM="github-actions-deployer@${PROJECT_ID}.iam.gserviceaccount.com"
export GITHUB_USERNAME="indracits"   # change to your actual GitHub username

echo "==============================================="
echo "Starting cleanup of all CI/CD resources..."
echo "Project: ${PROJECT_ID}"
echo "==============================================="

# -------------------------------
# 1. DELETE DOCKER IMAGES FROM ARTIFACT REGISTRY
# -------------------------------
echo "1. Deleting Docker images from Artifact Registry..."
gcloud artifacts docker images delete ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/client --delete-tags --quiet 2>/dev/null || echo " - No client images found."
gcloud artifacts docker images delete ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/server --delete-tags --quiet 2>/dev/null || echo " - No server images found."

# -------------------------------
# 2. DELETE ARTIFACT REGISTRY REPOSITORY
# -------------------------------
echo "2. Deleting Artifact Registry repository..."
gcloud artifacts repositories delete ${REPO_NAME} --location=${REGION} --quiet 2>/dev/null || echo "Repository not found."

# -------------------------------
# 3. DELETE WORKLOAD IDENTITY POOL AND PROVIDER
# -------------------------------
echo "3. Deleting Workload Identity Provider and Pool..."
gcloud iam workload-identity-pools providers delete ${WIF_PROVIDER} --location="global" --workload-identity-pool=${WIF_POOL} --quiet 2>/dev/null || echo "Provider not found."
gcloud iam workload-identity-pools delete ${WIF_POOL} --location="global" --quiet 2>/dev/null || echo "Pool not found."

# -------------------------------
# 4. REMOVE IAM BINDINGS FROM DEFAULT COMPUTE ENGINE SERVICE ACCOUNT
# (keep the SA itself – it's default)
# -------------------------------
echo "4. Removing IAM bindings from default Compute Engine service account..."

# Remove wildcard binding (if any)
gcloud iam service-accounts remove-iam-policy-binding ${SERVICE_ACCOUNT_DEFAULT} \
  --role="roles/iam.serviceAccountTokenCreator" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_ID}/locations/global/workloadIdentityPools/${WIF_POOL}/*" \
  --quiet 2>/dev/null || echo " - No wildcard binding."

# Remove specific GitHub user binding for TokenCreator
gcloud iam service-accounts remove-iam-policy-binding ${SERVICE_ACCOUNT_DEFAULT} \
  --role="roles/iam.serviceAccountTokenCreator" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_ID}/locations/global/workloadIdentityPools/${WIF_POOL}/attribute.repository_owner/${GITHUB_USERNAME}" \
  --quiet 2>/dev/null || echo " - No TokenCreator binding for ${GITHUB_USERNAME}."

# Remove workloadIdentityUser binding
gcloud iam service-accounts remove-iam-policy-binding ${SERVICE_ACCOUNT_DEFAULT} \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_ID}/locations/global/workloadIdentityPools/${WIF_POOL}/attribute.repository_owner/${GITHUB_USERNAME}" \
  --quiet 2>/dev/null || echo " - No workloadIdentityUser binding."

# -------------------------------
# 5. (OPTIONAL) DELETE CUSTOM SERVICE ACCOUNT IF EXISTS
# -------------------------------
echo "5. Deleting custom service account (if created)..."
gcloud iam service-accounts delete ${SERVICE_ACCOUNT_CUSTOM} --quiet 2>/dev/null || echo "Custom service account not found."

# -------------------------------
# 6. DELETE COMPUTE ENGINE VM
# -------------------------------
echo "6. Deleting Compute Engine VM..."
gcloud compute instances delete ${INSTANCE_NAME} --zone=${ZONE} --quiet 2>/dev/null || echo "VM already deleted or not found."

# -------------------------------
# 7. DELETE CLOUD SQL INSTANCE
# -------------------------------
echo "7. Deleting Cloud SQL instance..."
gcloud sql instances delete ${SQL_INSTANCE} --quiet 2>/dev/null || echo "Cloud SQL instance already deleted or not found."

# -------------------------------
# 8. REMOVE VPC PEERING
# -------------------------------
echo "8. Removing VPC peering..."
gcloud services vpc-peerings delete --service=servicenetworking.googleapis.com --network=default --quiet 2>/dev/null || echo "Peering already removed or not found."

# -------------------------------
# 9. DELETE RESERVED IP RANGE FOR PEERING
# -------------------------------
echo "9. Deleting reserved IP range for peering..."
gcloud compute addresses delete google-managed-services-default --global --quiet 2>/dev/null || echo "IP range already deleted or not found."

# -------------------------------
# 10. DELETE FIREWALL RULES
# -------------------------------
echo "10. Deleting firewall rules..."
for rule in allow-http allow-https allow-https-cicd-vm allow-https-restricted; do
    gcloud compute firewall-rules delete ${rule} --quiet 2>/dev/null || echo " - Rule ${rule} not found."
done

# -------------------------------
# 11. DELETE STATIC EXTERNAL IP (if reserved)
# -------------------------------
echo "11. Deleting static external IP (if any)..."
gcloud compute addresses delete cicd-vm-ip --region=${REGION} --quiet 2>/dev/null || echo "Static IP already deleted or not found."

echo "==============================================="
echo "Cleanup completed."
echo "==============================================="
```