# Set Up and Configure a Cloud Environment in Google Cloud: Challenge Lab (GSP321)

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP321-34A853)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-21759B?logo=wordpress&logoColor=white)

Reference solution for the Google Cloud challenge lab **Set Up and Configure a Cloud Environment in Google Cloud** (GSP321).

## 📖 Overview

You build the "Griffin" environment from scratch: development and production VPCs, a bastion host, a Cloud SQL database and a GKE cluster running WordPress through the Cloud SQL Auth Proxy.

## 🧰 Services used

VPC networks · Deployment Manager · Compute Engine · Cloud SQL for MySQL · Google Kubernetes Engine · Cloud Monitoring

## ✅ Prerequisites

```bash
export REGION="us-east1"
export ZONE="us-east1-b"
gcloud config set compute/zone $ZONE
```

## 🚀 Solution

### Task 1: Create the development VPC

Create `griffin-dev-vpc` with two subnets as listed in the lab:

```bash
gcloud compute networks create griffin-dev-vpc --subnet-mode custom

gcloud compute networks subnets create griffin-dev-wp \
  --network griffin-dev-vpc --region $REGION --range 192.168.16.0/20

gcloud compute networks subnets create griffin-dev-mgmt \
  --network griffin-dev-vpc --region $REGION --range 192.168.32.0/20
```

### Task 2: Create the production VPC with Deployment Manager

```bash
gsutil cp -r gs://cloud-training/gsp321/dm .
cd dm
sed -i "s/SET_REGION/$REGION/g" prod-network.yaml
gcloud deployment-manager deployments create prod-network --config=prod-network.yaml
cd ..
```

### Task 3: Create the bastion host

```bash
gcloud compute instances create bastion \
  --network-interface=network=griffin-dev-vpc,subnet=griffin-dev-mgmt \
  --network-interface=network=griffin-prod-vpc,subnet=griffin-prod-mgmt \
  --tags=ssh \
  --zone=$ZONE

gcloud compute firewall-rules create fw-ssh-dev \
  --network=griffin-dev-vpc --allow=tcp:22 --source-ranges=0.0.0.0/0 --target-tags=ssh

gcloud compute firewall-rules create fw-ssh-prod \
  --network=griffin-prod-vpc --allow=tcp:22 --source-ranges=0.0.0.0/0 --target-tags=ssh
```

### Task 4: Create and configure the Cloud SQL instance

```bash
gcloud sql instances create griffin-dev-db --region=$REGION --root-password="<root-password>"
gcloud sql connect griffin-dev-db
```

At the MySQL prompt:

```sql
CREATE DATABASE wordpress;
CREATE USER 'wp_user'@'%' IDENTIFIED BY 'stormwind_rules';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wp_user'@'%';
FLUSH PRIVILEGES;
```

### Task 5: Create the GKE cluster

```bash
gcloud container clusters create griffin-dev \
  --network griffin-dev-vpc \
  --subnetwork griffin-dev-wp \
  --machine-type n1-standard-4 \
  --num-nodes 2 \
  --zone $ZONE

gcloud container clusters get-credentials griffin-dev --zone $ZONE
```

### Task 6: Prepare the Kubernetes cluster

```bash
cd ~
gsutil cp -r gs://cloud-training/gsp321/wp-k8s .
cd wp-k8s

sed -i 's/username_goes_here/wp_user/g'         wp-env.yaml
sed -i 's/password_goes_here/stormwind_rules/g' wp-env.yaml
kubectl create -f wp-env.yaml

gcloud iam service-accounts keys create key.json \
  --iam-account=cloud-sql-proxy@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
kubectl create secret generic cloudsql-instance-credentials --from-file key.json
```

### Task 7: Create the WordPress deployment

```bash
INSTANCE_CONNECTION=$(gcloud sql instances describe griffin-dev-db --format="value(connectionName)")
sed -i "s/YOUR_SQL_INSTANCE/$INSTANCE_CONNECTION/g" wp-deployment.yaml

kubectl create -f wp-deployment.yaml
kubectl create -f wp-service.yaml
kubectl get service wordpress --watch   # wait for EXTERNAL-IP
```

### Task 8: Enable monitoring

In **Cloud Monitoring → Uptime checks**, create an HTTP uptime check against the WordPress external IP.

### Task 9: Grant access to an additional engineer

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="user:<second-user-email>" \
  --role="roles/editor"
```

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
