# GCP TEE Provisioning - Required IAM Roles

This document describes the GCP IAM roles required for TEE provisioning.

## Overview

GCP uses IAM roles that you assign to a Service Account. The Agency app uses these credentials to provision Confidential VMs in your GCP project.

---

## Option 1: Use Predefined Roles (Recommended for Simplicity)

Assign these predefined roles to a Service Account:

| Role | Purpose |
|------|---------|
| `roles/compute.instanceAdmin.v1` | Create, manage, delete Confidential VM instances |
| `roles/compute.securityAdmin` | Manage firewall rules and security policies |
| `roles/iam.serviceAccountUser` | Attach service accounts to VMs |
| `roles/storage.objectViewer` | Read from GCS buckets (for direct ingestion) |

### gcloud CLI Commands

```bash
# Variables
PROJECT_ID="your-project-id"
SERVICE_ACCOUNT_EMAIL="agency-tee@${PROJECT_ID}.iam.gserviceaccount.com"

# Create service account
gcloud iam service-accounts create agency-tee \
    --project="$PROJECT_ID" \
    --display-name="Agency TEE Provisioner"

# Assign roles
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role="roles/compute.instanceAdmin.v1"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role="roles/compute.securityAdmin"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role="roles/iam.serviceAccountUser"

# Optional: For direct ingestion from GCS
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role="roles/storage.objectViewer"
```

---

## Option 2: Custom Role (Least Privilege)

For maximum security, create a custom role with only the required permissions:

### Custom Role Definition

```yaml
title: "Agency TEE Provisioner"
description: "Custom role for Agency TEE provisioning with least privilege"
stage: "GA"
includedPermissions:
  # Compute Instance Management
  - compute.instances.create
  - compute.instances.delete
  - compute.instances.get
  - compute.instances.list
  - compute.instances.setMetadata
  - compute.instances.setTags
  - compute.instances.setLabels
  - compute.instances.start
  - compute.instances.stop
  - compute.instances.reset
  - compute.instances.setMachineType
  - compute.instances.attachDisk
  - compute.instances.detachDisk
  
  # Disk Management
  - compute.disks.create
  - compute.disks.delete
  - compute.disks.get
  - compute.disks.list
  - compute.disks.use
  - compute.disks.setLabels
  
  # Snapshot Management
  - compute.snapshots.create
  - compute.snapshots.delete
  - compute.snapshots.get
  - compute.snapshots.list
  - compute.snapshots.useReadOnly
  
  # Networking
  - compute.networks.get
  - compute.networks.list
  - compute.subnetworks.get
  - compute.subnetworks.list
  - compute.subnetworks.use
  - compute.firewalls.create
  - compute.firewalls.delete
  - compute.firewalls.get
  - compute.firewalls.list
  - compute.firewalls.update
  - compute.addresses.create
  - compute.addresses.delete
  - compute.addresses.get
  - compute.addresses.list
  - compute.addresses.use
  
  # Zones and Machine Types
  - compute.zones.get
  - compute.zones.list
  - compute.machineTypes.get
  - compute.machineTypes.list
  
  # Images
  - compute.images.get
  - compute.images.list
  - compute.images.useReadOnly
  
  # Service Account Usage
  - iam.serviceAccounts.actAs
  
  # Storage (for direct ingestion)
  - storage.objects.get
  - storage.objects.list
  - storage.buckets.get
```

### Create Custom Role via gcloud

```bash
# Save the YAML above to agency-tee-role.yaml, then:
gcloud iam roles create AgencyTEEProvisioner \
    --project="$PROJECT_ID" \
    --file=agency-tee-role.yaml

# Assign to service account
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member="serviceAccount:$SERVICE_ACCOUNT_EMAIL" \
    --role="projects/$PROJECT_ID/roles/AgencyTEEProvisioner"
```

---

## Required Credentials for Agency App

When configuring GCP in the Agency app, you'll need:

| Field | Description | How to Get |
|-------|-------------|------------|
| **Service Account JSON** | Full JSON key file | Download from GCP Console or gcloud |
| **Project ID** | GCP project identifier | In the JSON file or GCP Console |
| **Zone** | Default zone for VMs | e.g., `us-central1-a` |

### Generate Service Account Key

```bash
# Create and download key
gcloud iam service-accounts keys create agency-tee-key.json \
    --iam-account="$SERVICE_ACCOUNT_EMAIL" \
    --project="$PROJECT_ID"

# The JSON file contains all required credentials
cat agency-tee-key.json
```

**Security Note:** Treat this JSON file as a secret. It provides full access per the assigned roles.

---

## Confidential VM Requirements

For AMD SEV Confidential VMs, additional considerations:

| Requirement | Details |
|-------------|---------|
| **Machine Type** | Must be N2D series (e.g., `n2d-standard-2`, `n2d-standard-4`) |
| **Regions/Zones** | Most regions support Confidential VMs; check [availability](https://cloud.google.com/compute/docs/regions-zones) |
| **Image** | Must use a Confidential VM compatible OS image |
| **CPU Platform** | AMD Milan or later |

### Verify Confidential VM Support

```bash
# List available N2D machine types in a zone
gcloud compute machine-types list \
    --filter="zone:us-central1-a AND name~'n2d'" \
    --project="$PROJECT_ID"

# Check if Confidential Computing API is enabled
gcloud services list --enabled --project="$PROJECT_ID" | grep confidential
```

---

## Enable Required APIs

Before provisioning, ensure these APIs are enabled:

```bash
gcloud services enable compute.googleapis.com --project="$PROJECT_ID"
gcloud services enable confidentialcomputing.googleapis.com --project="$PROJECT_ID"
gcloud services enable iam.googleapis.com --project="$PROJECT_ID"
gcloud services enable storage.googleapis.com --project="$PROJECT_ID"
```

---

## Verification

Test your credentials:

```bash
# Activate service account
gcloud auth activate-service-account --key-file=agency-tee-key.json

# Set project
gcloud config set project "$PROJECT_ID"

# Verify compute access
gcloud compute instances list

# Check Confidential VM capability
gcloud compute instances create test-confidential \
    --zone=us-central1-a \
    --machine-type=n2d-standard-2 \
    --confidential-compute \
    --maintenance-policy=TERMINATE \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --dry-run

# Clean up (if you ran without --dry-run)
# gcloud compute instances delete test-confidential --zone=us-central1-a
```
