# 🚀 POC: Secure and Automated AKS Cloud Platform

> **Document Version:** 1.2  
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
## 🛣️ Project Phases and Next Steps

This project will be executed in distinct phases, moving from a foundational manual setup to a fully automated, scalable platform. This approach ensures a controlled rollout and sets clear expectations.

* **Phase 1: Foundational Bootstrap (Manual Setup)**
    * **Objective:** To solve the initial "chicken-and-egg" problem by creating the absolute minimum resources required to enable automation.
    * **Tasks:**
        * Manually create the **Service Principal** (App Registration) in Azure AD to act as the identity for our CI/CD pipeline.
        * Manually create the **Azure Storage Account** that will securely store the Terraform state file.
    * **Outcome:** A secure foundation for all subsequent automated work. This phase is performed once and is not part of the recurring workflow.

* **Phase 2: Core Infrastructure Automation (This POC)**
    * **Objective:** To build and validate the core, automated infrastructure platform as defined in this document's scope.
    * **Tasks:**
        * Develop reusable Terraform modules for networking, AKS, Bastion, and the CI/CD runner.
        * Create a GitHub Actions pipeline that uses the bootstrapped identity to deploy the infrastructure.
        * Deploy a sample application to verify end-to-end functionality.
    * **Outcome:** A working, automated, and secure AKS platform that meets all success criteria.

* **Phase 3: Cluster Onboarding & Hardening (Next Steps)**
    * **Objective:** To prepare the validated platform for its first real workloads by adding essential services and governance.
    * **Potential Tasks:**
        * Automate the deployment of an NGINX Ingress Controller for traffic management.
        * Integrate with Azure Monitor for basic logging and metrics.
        * Implement Azure Policy to enforce security and compliance standards (e.g., prevent public load balancers).
        * Create a standardized onboarding guide for the first pilot application team.

* **Phase 4: Full GitOps & Self-Service (Future Vision)**
    * **Objective:** To evolve the platform into a true self-service model for application deployments.
    * **Potential Tasks:**
        * Implement a GitOps controller (like ArgoCD or Flux) for continuous application deployment.
        * Develop standardized patterns for stateful applications using Azure Files/Disk.
        * Publish a service catalog of reusable Terraform modules for development teams.

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
    1.  **Azure Private DNS Zone:** We will create a private zone for our domain.
    2.  **VNet Link:** This zone will be linked to our VNet, making it authoritative for all resources within it.
    3.  **ExternalDNS Controller:** We will deploy the ExternalDNS controller to our AKS cluster to automatically create `A` records in our private zone from Kubernetes Ingress resources.
    4.  **Conditional Forwarding:** The on-premises network team will configure the corporate DNS server to forward all queries for `hero.org` to Azure's internal DNS resolver (`168.63.129.16`).
 
---

## 🏛️ Terraform Code Structure

The Terraform project will be organized using a modular approach to promote reusability, maintainability, and consistency. The directory structure is designed to separate the reusable "building blocks" (`modules`) from the environment-specific "implementations" (`live`).
