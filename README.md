# 🚀 POC: Secure and Automated AKS Cloud Platform

> **Document Version:** 1.1  
> **Status:** Proposed  
> **Owner:** DevOps Engineering  
> **Date:** October 5, 2025

---
## Executive Summary

This document outlines a Proof of Concept (POC) to validate a secure, automated, and production-ready architectural pattern for deploying applications on the Azure Kubernetes Service (AKS). The primary objective is to demonstrate the feasibility of managing the entire infrastructure lifecycle via a GitOps workflow, leveraging Infrastructure as Code (IaC) principles. The successful completion of this POC will establish a standardized blueprint for migrating future workloads to a modern, scalable, and secure cloud-native platform, ensuring seamless and secure integration with existing corporate network and identity services.

---
## 🎯 Business Objective & Problem Statement

### Objective
To increase development velocity, enhance security posture, and improve operational efficiency by creating a standardized, automated platform for cloud-native applications.

### Problem Statement
The current process for deploying cloud infrastructure can be manual and inconsistent, leading to slower delivery cycles, potential security vulnerabilities, and configuration drift. A lack of a standardized, automated platform creates operational overhead and hinders our ability to scale effectively. This POC addresses the need for a secure, repeatable, and auditable process for infrastructure provisioning and application deployment.

---
## 🗺️ Scope of Work

| In-Scope (Deliverables) | Out-of-Scope (Items) |
| :--- | :--- |
| ✅ Automated provisioning of all Azure networking components via Terraform. | ❌ Deployment of complex, multi-tier stateful applications. |
| ✅ Deployment of a secure administrative access method (Azure Bastion). | ❌ Performance benchmarking, load testing, or formal DR testing. |
| ✅ Deployment and configuration of a private self-hosted GitHub Actions runner. | ❌ Implementation of advanced observability (monitoring, logging, tracing). |
| ✅ Automated deployment of a private AKS cluster using the self-hosted runner. | ❌ Detailed cost analysis and financial optimization reporting. |
| ✅ Configuration and validation of the Hybrid DNS resolution flow. | |
| ✅ Deployment of a sample stateless application to verify the end-to-end workflow. | |

---
## 🏗️ Technical Architecture Deep Dive

This section details the specific technical configurations and decisions for the core components of the architecture.

* **VNet and Subnet Address Space:**
    * **Virtual Network (VNet):** `10.50.0.0/16` - Provides 65,536 total IP addresses, offering ample room for future expansion.
    * **Subnets:**
        * `AKSSubnet`: `10.50.0.0/23` - Allocates 512 IPs. This supports the Azure CNI requirement for a unique IP per pod and can comfortably host a scaled-out cluster (e.g., up to 15 nodes with the default 30 pods per node).
        * `RunnerSubnet`: `10.50.2.0/27` - Allocates 32 IPs, sufficient for a small pool of CI/CD runner VMs.
        * `AzureBastionSubnet`: `10.50.3.0/27` - Allocates 32 IPs. Microsoft recommends at least a `/27` for this dedicated subnet.

* **AKS Network Profile & Policy:**
    * **Network Profile:** We will use **Azure CNI**. This high-performance network plugin assigns a full VNet IP address to each pod, enabling direct connectivity and compatibility with advanced network features.
    * **Network Policy:** We will enable **Azure Network Policy** (or alternatively, Calico). This acts as a firewall within our cluster, allowing us to define rules that control which pods can communicate with each other, significantly enhancing intra-cluster security.

* **Storage Configuration:**
    * **Terraform State File:** The Terraform state file will be stored securely in an **Azure Blob Storage** container, a separate and dedicated resource for this critical function.
    * **AKS Pod Storage:** For this POC's stateless application, we will use the default ephemeral storage on the nodes. Future stateful applications will use **Azure Disk** (for single-pod, read/write volumes) or **Azure Files** (for shared storage).

* **Identity, Access & Permissions:**
    * **CI/CD Pipeline Identity:** The GitHub Actions workflow will use an **App Registration** with a **Federated Credential (OIDC)** for a modern, secretless authentication approach.
    * **AKS Cluster Identity:** The AKS cluster itself will be assigned a **System-Assigned Managed Identity** to interact with other Azure resources on its behalf (e.g., creating load balancers or attaching disks).

* **DNS Implementation Strategy:**
    1.  **Azure Private DNS Zone:** We will create a private zone for our domain.
    2.  **VNet Link:** This zone will be linked to our VNet, making it authoritative for all resources within it.
    3.  **ExternalDNS Controller:** We will deploy the ExternalDNS controller to our AKS cluster to automatically create `A` records in our private zone from Kubernetes Ingress resources.
    4.  **Conditional Forwarding:** The on-premises network team will configure the corporate DNS server to forward all queries for `hero.org` to Azure's internal DNS resolver (`168.63.129.16`).

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
