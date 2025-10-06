# 🧩 POC: Secure and Automated AKS Cloud Platform

**Document Version:** 1.2  
**Status:** Proposed  
**Owner:** DevOps Engineering  
**Date:** October 5, 2025  

---

## 📝 Executive Summary

This document outlines a Proof of Concept (POC) to validate a secure, automated, and production-ready architectural pattern for deploying applications on the **Azure Kubernetes Service (AKS)**.  
The primary objective is to demonstrate the feasibility of managing the entire infrastructure lifecycle via a **GitOps workflow**, leveraging **Infrastructure as Code (IaC)** principles.  

The successful completion of this POC will establish a standardized blueprint for migrating future workloads to a modern, scalable, and secure cloud-native platform, ensuring seamless and secure integration with existing corporate network and identity services.

---

## 🎯 Business Objective & Problem Statement

### **Objective**
To increase development velocity, enhance security posture, and improve operational efficiency by creating a standardized, automated platform for cloud-native applications.

### **Problem Statement**
The current process for deploying cloud infrastructure can be manual and inconsistent, leading to slower delivery cycles, potential security vulnerabilities, and configuration drift.  
A lack of a standardized, automated platform creates operational overhead and hinders our ability to scale effectively.  
This POC addresses the need for a secure, repeatable, and auditable process for infrastructure provisioning and application deployment.

---

## 🗺️ Scope of Work

| **In-Scope (Deliverables)** | **Out-of-Scope (Items)** |
|------------------------------|---------------------------|
| ✅ Automated provisioning of all Azure networking components via Terraform. | ❌ Deployment of complex, multi-tier stateful applications. |
| ✅ Deployment of a secure administrative access method (Azure Bastion). | ❌ Performance benchmarking, load testing, or formal DR testing. |
| ✅ Deployment and configuration of a private self-hosted GitHub Actions runner. | ❌ Implementation of advanced observability (monitoring, logging, tracing). |
| ✅ Automated deployment of a private AKS cluster using the self-hosted runner. | ❌ Detailed cost analysis and financial optimization reporting. |
| ✅ Configuration and validation of the Hybrid DNS resolution flow. |   |
| ✅ Deployment of a sample stateless application to verify the end-to-end workflow. |   |

---

## 🛣️ Project Phases and Next Steps

This project will be executed in distinct phases, moving from a foundational manual setup to a fully automated, scalable platform.

### **Phase 1: Foundational Bootstrap (Manual Setup)**
**Objective:** Solve the “chicken-and-egg” problem by creating the minimum resources required to enable automation.

**Tasks:**
- Manually create the Service Principal (App Registration) in Azure AD to act as the CI/CD pipeline identity.  
- Manually create the Azure Storage Account to securely store the Terraform state file.  

**Outcome:** A secure foundation for all subsequent automated work.

---

### **Phase 2: Core Infrastructure Automation (This POC)**
**Objective:** Build and validate the core, automated infrastructure platform as defined in this document.

**Tasks:**
- Develop reusable Terraform modules for networking, AKS, Bastion, and the CI/CD runner.  
- Create a GitHub Actions pipeline using the bootstrapped identity to deploy infrastructure.  
- Deploy a sample application to verify end-to-end functionality.  

**Outcome:** A working, automated, and secure AKS platform.

---

### **Phase 3: Cluster Onboarding & Hardening (Next Steps)**
**Objective:** Prepare the validated platform for production workloads.

**Potential Tasks:**
- Automate NGINX Ingress Controller deployment.  
- Integrate with Azure Monitor.  
- Implement Azure Policy for compliance.  
- Create onboarding guide for pilot application teams.  

---

### **Phase 4: Full GitOps & Self-Service (Future Vision)**
**Objective:** Evolve the platform into a true self-service model.

**Potential Tasks:**
- Implement GitOps (ArgoCD/Flux).  
- Develop stateful app patterns (Azure Files/Disk).  
- Publish a reusable Terraform service catalog.  

---

## Architecture


<img width="3368" height="3276" alt="image" src="https://github.com/user-attachments/assets/94ecafe9-ace2-45f7-bd87-ac233f3d953d" />


---

## 🏗️ Technical Architecture Deep Dive

### **VNet and Subnet Address Space**
- **VNet:** `10.50.0.0/16` — 65,536 IPs  
- **Subnets:**
  - `AKSSubnet: 10.50.0.0/23` — 512 IPs for pods and nodes  
  - `RunnerSubnet: 10.50.2.0/27` — 32 IPs for CI/CD runners  
  - `AzureBastionSubnet: 10.50.3.0/27` — 32 IPs, per Microsoft recommendation  

### **AKS Network Profile & Policy**
- **Network Plugin:** Azure CNI  
- **Network Policy:** Azure Network Policy (or Calico) for intra-cluster firewalling.  

### **Storage Configuration**
- **Terraform State File:** Stored in Azure Blob Storage.  
- **AKS Pod Storage:** Default ephemeral storage; future use of Azure Disks or Files.  

### **Identity, Access & Permissions**
- **CI/CD Identity:** GitHub Actions with Federated OIDC for secretless auth.  
- **AKS Cluster Identity:** System-assigned Managed Identity for resource access.  

### **DNS Implementation Strategy**
1. Create Azure Private DNS Zone.  
2. Link zone to VNet.  
3. Deploy ExternalDNS controller in AKS.  
4. Configure on-prem DNS for conditional forwarding.  

---

## 🏛️ Terraform Code Structure

```plaintext
terraform-aks-platform/
├── modules/                # Reusable components
│   ├── networking/         # VNet, subnets, NSGs
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── aks/                # AKS cluster & node pools
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── bastion/            # Azure Bastion host
│   │   ├── main.tf
│   │   └── variables.tf
│   └── runner-vm/          # GitHub Actions runner VM
│       ├── main.tf
│       └── variables.tf
└── live/                   # Environment-specific configs
    └── prod/
        ├── main.tf
        ├── terraform.tfvars
        └── backend.tf
```

* **Sizing and VM Selection:**

| Component | Recommended Size | Rationale |
| :--- | :--- | :--- |
| **AKS System Nodepool** | 2 x `Standard_D2s_v5` | System pods are lightweight. Two small, cost-effective nodes provide high availability for critical cluster services. |
| **AKS User Nodepool** | 2 x `Standard_D4s_v5` | A solid general-purpose starting point (4 vCPUs, 16 GiB RAM). We will enable the cluster autoscaler to scale this nodepool up to 10 nodes based on demand. |
| **Self-Hosted Runner VM**| 1 x `Standard_D2s_v5` | CI/CD jobs are bursty. This size (2 vCPUs, 8 GiB RAM) provides a good balance of CPU and fast Premium SSD storage for running Terraform and build jobs efficiently. |
| **Azure Bastion Host** | `Standard` SKU | The Standard SKU is required for features like scaling and is recommended for production environments. Sizing is managed by Azure. |

* **Linux Profile (Node Configuration):**
    We will use the default Azure-tuned Ubuntu image for our AKS nodes. We will configure SSH key access to the nodes during creation, which, combined with Azure Bastion, will allow for secure "break-glass" debugging if ever required.

---
## 💼 Strategic Importance & Business Value

This architecture is a strategic enabler for the business, providing:
* **Enhanced Security Posture:** A "zero-trust" network with private endpoints and fine-grained network policies drastically reduces the attack surface.
* **Increased Development Velocity:** The GitOps workflow turns the deployment process into a simple `git push`, removing manual bottlenecks and accelerating time-to-market.
* **Operational Excellence:** Infrastructure as Code eliminates configuration drift and makes our platform repeatable, auditable, and easy to recover.
* **Scalability & Cost-Effectiveness:** The use of the cluster autoscaler ensures that we only pay for the compute resources we need, automatically scaling to meet user demand.

---
## ✅ Success Criteria

The Proof of Concept will be deemed successful upon the unconditional achievement of the following criteria:
* **Automation:** All Azure infrastructure is successfully provisioned and modified exclusively through the GitHub Actions pipeline triggered by a `git push`.
* **Security:** The deployed AKS cluster's API server is confirmed to be private and not accessible from the public internet.
* **Integration:** A DNS query for a deployed application's hostname from the corporate network resolves correctly to the private IP of the Ingress Controller.
* **Functionality:** The sample application is successfully deployed and is fully accessible from a client on the corporate network.

---
## 👥 Required Resources & Stakeholders

* **Azure Platform:**
    * An active Azure Subscription with Contributor-level permissions.
    * Sufficient vCPU quota for the planned VM and AKS node pools.
* **Personnel & Stakeholders:**
    * **DevOps Team:** Project lead and primary executors of the POC.
    * **Network Engineering Team:** Required stakeholder for the configuration of the on-premises DNS conditional forwarder.
    * **Security & Compliance Team:** Stakeholder for architectural review and approval.







