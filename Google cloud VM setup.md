

```bash

#!/bin/bash
# ============================================================
# Low‑cost CI/CD: private Cloud SQL + VM (password auth)
# - Cloud SQL has only a private IP.
# - VM has a public IP for SSH, but connects to DB privately.
# ============================================================

export PROJECT_ID=$(gcloud config get-value project)
export REGION="us-central1"
export ZONE="us-central1-a"
export INSTANCE_NAME="cicd-vm"
export SQL_INSTANCE="cicd-db"
export DB_NAME="cicddb"
export DB_USER="cicduser"
export VPC_NETWORK="default"

# Generate a random database password
DB_PASS=$(openssl rand -base64 12)

echo "==============================================="
echo "1. Setting up private services access..."
echo "==============================================="

# Step 1. Allocate a private IP range for services (if not already done)
# We choose 172.16.0.0/16 – adjust if it conflicts with existing subnets.
gcloud compute addresses create google-managed-services-default \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --network=${VPC_NETWORK} \
    --description="Peering range for Cloud SQL" \
    --quiet 2>/dev/null || echo "Address range may already exist, continuing..."

# Step 2. Create the private services connection (peering)
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default \
    --network=${VPC_NETWORK} \
    --quiet 2>/dev/null || echo "Peering may already exist, continuing..."

echo ""
echo "2. Creating Cloud SQL instance (private IP only)..."

# Step 3. Create the Cloud SQL instance with no public IP
gcloud sql instances create ${SQL_INSTANCE} \
    --database-version=MYSQL_8_0 \
    --tier=db-f1-micro \
    --region=${REGION} \
    --storage-size=10 \
    --storage-type=SSD \
    --backup-start-time=03:00 \
    --no-backup \
    --availability-type=zonal \
    --network=${VPC_NETWORK} \
    --no-assign-ip \
    --quiet

echo ""
echo "3. Creating database and user..."

# Step 4. Create database and a classic user
gcloud sql databases create ${DB_NAME} --instance=${SQL_INSTANCE} --quiet

gcloud sql users create ${DB_USER} \
    --instance=${SQL_INSTANCE} \
    --password="${DB_PASS}"

echo ""
echo "4. Creating Compute Engine VM..."

# Step 5. Create the VM (public IP for SSH, no extra startup script needed)
gcloud compute instances create ${INSTANCE_NAME} \
    --zone=${ZONE} \
    --machine-type=e2-micro \
    --boot-disk-size=30 \
    --boot-disk-type=pd-standard \
    --image-family=ubuntu-2204-lts \
    --image-project=ubuntu-os-cloud \
    --network=${VPC_NETWORK} \
    --scopes=cloud-platform \
    --tags=cicd-vm \
    --quiet

# Step 6. Retrieve IPs
SQL_PRIVATE_IP=$(gcloud sql instances describe ${SQL_INSTANCE} \
    --format="value(ipAddresses[0].ipAddress)")

VM_PUBLIC_IP=$(gcloud compute instances describe ${INSTANCE_NAME} \
    --zone=${ZONE} \
    --format="get(networkInterfaces[0].accessConfigs[0].natIP)")

# Step 7. Print connection details
echo ""
echo "==============================================="
echo "Environment is ready (private database access)."
echo "==============================================="
echo "VM external IP (for SSH):   ${VM_PUBLIC_IP}"
echo "Cloud SQL private IP:       ${SQL_PRIVATE_IP}"
echo "Database name:              ${DB_NAME}"
echo "Database user:              ${DB_USER}"
echo "Database password:          ${DB_PASS}"
echo ""
echo "SSH into the VM:"
echo "  gcloud compute ssh ${INSTANCE_NAME} --zone=${ZONE}"
echo ""
echo "Once inside the VM, install mysql-client and connect:"
echo "  sudo apt update && sudo apt install -y mysql-client"
echo "  mysql -u ${DB_USER} -p -h ${SQL_PRIVATE_IP}"
echo ""
echo "⚠️  Save the password securely – it won’t be shown again."
echo "==============================================="




#test the setup

gcloud compute ssh cicd-vm --zone=us-central1-a
sudo apt update && sudo apt install -y mysql-client
mysql -u cicduser -p -h <private_ip_of_cloud_sql>   # enter the password



#clean up
gcloud compute instances delete cicd-vm --zone=us-central1-a --quiet
gcloud sql instances delete cicd-db --quiet
gcloud compute addresses delete google-managed-services-default --global --quiet
# (also remove the peering if you created it, but it can be left unused)

```