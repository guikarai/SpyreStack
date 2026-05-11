# ACC and SSA Backup and Restore Documentation
## Spyre Stack Installation on IBM Z (LinuxONE)

---

**Version:** 1.0  
**Date:** May 2026  
**Author:** IBM Z Technical Documentation  
**Target Audience:** System Administrators, Infrastructure Engineers

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Backup and Restore Options Overview](#2-backup-and-restore-options-overview)
3. [ACC and SSA Configuration Backup and Restore](#3-acc-ssa-configuration-backup-and-restore)


---

## 1. Introduction

### 1.1 Context

This document describes the backup and restore procedures for **ACC (Appliance Control Center)** and **SSA (Spyre Support Appliance)** components as part of a Spyre Stack installation on IBM Z (LinuxONE).

ACC and SSAs are critical components of the IBM Z infrastructure that may require appropriate backup strategies to ensure business continuity and disaster recovery capabilities.

### 1.2 Document Objectives

This document aims to:
- Explain different backup and restore approaches
- Provide detailed procedures for each method
- Guide administrators in choosing the appropriate method
- Ensure compliance with IBM best practices

### 1.3 Prerequisites

Before proceeding with backup/restore operations, ensure you have:

- ✅ Administrative access to SSC (Secure Service Container) LPARs
- ✅ Access to HMC (Hardware Management Console)
- ✅ Knowledge of IBM Z architecture and ACC/SSA components
- ✅ Sufficient storage space for backups
- ✅ Documentation of current configuration
- ✅ Planned maintenance window (for certain operations)

### 1.4 References

- [IBM Documentation - Creating an ACC restore checkpoint](https://www.ibm.com/docs/en/systems-hardware/linuxone/9175-ML1?topic=linuxone-creating-acc-restore-checkpoint)

---

## 2. Backup and Restore Options Overview

There are **two main approaches** for backing up and restoring ACC and SSA configurations:

### 2.1 Approach 1: Disk-Level Copy (Raw Level)

This method involves performing a complete copy of storage resources attached to SSC LPARs.

**Characteristics:**
- Bit-by-bit copy of storage volumes
- Complete system backup including OS, configurations, and data
- Fast recovery in case of hardware failure

### 2.2 Approach 2: Configuration Backup (Configuration Level)

This method focuses on backing up logical ACC and SSA configurations via checkpoints and configuration exports.

**Characteristics:**
- Selective configuration backup
- Portability between different systems
- Flexibility for partial restorations

### 2.3 Comparison Table

| Criteria | Disk-Level Copy | Configuration Backup |
|----------|-----------------|---------------------|
| **Granularity** | Complete (entire system) | Selective (configurations only) |
| **Backup Time** | Long (depends on volume size) | Short (few minutes) |
| **Restore Time** | Medium to long | Short |
| **Space Required** | Large (volume size) | Minimal (configuration files) |
| **Portability** | Limited (same hardware) | High (different systems) |
| **Complexity** | Medium | Low |
| **System Impact** | Requires shutdown or snapshot | Minimal (online operation) |
| **Use Case** | Complete Disaster Recovery | Migration, update, rollback |

### 2.4 Recommendations

**Use disk-level copy if:**
- You are preparing a complete Disaster Recovery plan
- You need to backup the entire system (OS + configurations + data)
- You have snapshot capabilities at storage level
- Backup time is not critical

**Use configuration backup if:**
- You are performing a migration to a new system
- You are preparing a major update with rollback capability
- You need to backup configurations frequently
- Storage space is limited
- You need portability between environments

---

## 3. ACC and SSA Configuration Backup and Restore

### 3.1 Introduction to ACC/SSA

You may need to revert IBM Z Appliance Control Center (ACC) or IBM Z Spyre Appliance Sypport (SSA) to a previous state or configuration. This may be necessary if an upgrade or incorrect input corrupts the ACC/SSA configuration. This requires the creation of ACC or SSA Restore Checkpoints.

To prepare for such scenarios, Spyre administrators should regularly create and securely store checkpoints of ACC/SSA configurations. You can automate this process by creating a job that collects and stores these checkpoints.

### 3.2 Creating an ACC/SSA Restore Checkpoint

#### 3.2.1 Prerequisites
You will use API of ACC and/or SSA to make it happened.
It requires to known and to fill in the following variables.
| Variable | Purpose |
|:-------- |:--------:|
| ACC_IP    | IP address of ACC   |
| ACC_LPAR_USERNAME     | Username of the SSC LPAR on the HMC profile (used during ACC install)   |
| ACC_LPAR_PASSWORD    | Password of the SSC LPAR on the HMC profile (used during ACC install)   |
| REASON    | Reason for capturing the ACC configuration   |

#### 3.2.2 To create an ACC Checkpoint

1. Log in to the Appliance. Generate an appliance admin token (SSC_TOKEN) using the SSC REST API:
```
curl -k -X 'POST' \
  "https://$ACC_IP/api/com.ibm.zaci.system/api-tokens" \
  -H 'accept: application/vnd.ibm.zaci.payload+json' \
  -H 'Content-Type: application/vnd.ibm.zaci.payload+json;version=1.0' \
  -H 'zACI-API: com.ibm.zaci.system/1.0' \
  -d '{
    "kind": "request",
    "parameters": {
      "user": "admin",
      "password": "P@55word"
    }
  }'
```

2. Store the returned token in SSC_TOKEN. This token is required for the next steps.

3. Export the Configuration. Run the following command to export the ACC configuration:
```
curl -k -X 'POST' \
  "https://$ACC_IP/api/com.ibm.zaci.system/appliance-configuration/export" \
  -H "Authorization: Bearer $SSC_TOKEN" \
  -H "zACI-API: com.ibm.zaci.system/1.0" \
  -H "Content-type: application/vnd.ibm.zaci.payload+json;version=1.0" \
  -H "Accept: application/octet-stream" \
  -o "acc-config-$(date +%d-%m-%Y).data" \
  -d '{
    "kind": "request", 
    "parameters": {
      "description": "'Create ACC configuration data'"
    }
  }'
```

This command:
* Creates a configuration snapshot on the ACC.
* Downloads the configuration file as acc-config-$(date +%d-%m-%Y).data.

**Note:** Important: Store this file securely.

#### 3.2.3 To restore an ACC Checkpoint

To restore ACC to a previous state, import the saved configuration:
```
curl -k -X 'POST' \
  "https://$ACC_IP/api/com.ibm.zaci.system/appliance-configuration/import?apply_now=true" \
  -H "Authorization: Bearer $SSC_TOKEN" \
  -H "zACI-API: com.ibm.zaci.system/1.0" \
  -H "Accept: application/vnd.ibm.zaci.payload+json" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @acc-config-25-09-2025.data
```

**Warning:** The appliance will restart after the configuration is applied.
**Note:** Currently, ACC configuration capture and restore is only supported within the same version of ACC. Cross-version configuration migration is not supported at this time. In addition, creating an ACC checkpoint does not copy or preserve appliance images and fixes that users have already uploaded to ACC. Appliance owners are therefore responsible for saving these files or downloading them again from Fix Central.
