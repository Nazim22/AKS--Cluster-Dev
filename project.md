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

---

## 🧮 Architecture Sizing and VM Selection

| **Component** | **Recommended Size** | **Rationale** |
|----------------|----------------------|----------------|
| **AKS System Nodepool** | 2 × Standard_D2s_v5 | Lightweight system pods, high availability. |
| **AKS User Nodepool** | 2 × Standard_D4s_v5 | General purpose; autoscaler enabled. |
| **Self-Hosted Runner VM** | 1 × Standard_D2s_v5 | Balanced for Terraform/CI/CD workloads. |
| **Azure Bastion Host** | Standard SKU | Required for production-grade features. |

---

### 🐧 Linux Profile (Node Configuration)

Using Azure-tuned **Ubuntu** images with **SSH key access** via **Azure Bastion** for secure, break-glass debugging and administrative access.

---

## 💼 Strategic Importance & Business Value

- **Enhanced Security Posture:** Implements zero-trust networking with private endpoints and fine-grained network policies to reduce the attack surface.  
- **Increased Development Velocity:** The GitOps model enables push-based deployments, removing manual bottlenecks.  
- **Operational Excellence:** Infrastructure as Code (IaC) ensures repeatability, auditability, and faster recovery.  
- **Scalability & Cost-Effectiveness:** Autoscaler dynamically adjusts resources, optimizing for cost and performance.  

---

## ✅ Success Criteria

The POC will be deemed successful upon meeting the following conditions:

- **Automation:** All Azure infrastructure is provisioned and updated via GitHub Actions pipelines.  
- **Security:** The AKS API server is private and inaccessible from the public internet.  
- **Integration:** DNS queries for deployed applications resolve correctly within the corporate network.  
- **Functionality:** The sample application is deployed successfully and accessible from internal clients.  

---

## 👥 Required Resources & Stakeholders

### **Azure Platform**
- Active **Azure Subscription** with Contributor-level permissions.  
- Adequate **vCPU quota** for AKS clusters, Bastion, and VM workloads.  

### **Personnel & Stakeholders**
- **DevOps Team:** Project leads and executors responsible for automation and implementation.  
- **Network Engineering Team:** Configure DNS conditional forwarders between corporate and Azure networks.  
- **Security & Compliance Team:** Review and approve architecture for alignment with corporate standards.  

