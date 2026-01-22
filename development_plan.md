# k0rdent Personal Development Plan

## 1. Foundational Skills — Existing Knowledge

### ~~YAML~~
**~~Theory~~**
- ~~Refresh YAML syntax and structure.~~
- ~~Review indentation rules, lists, and key–value pairs.~~

**~~Practice~~**
- ~~Write a Kubernetes manifest in YAML from scratch.~~
- ~~Validate it using a linter or `kubectl`.~~

---

### ~~Containerization~~
**~~Theory~~**
- ~~Review core containerization concepts: images and containers.~~

**~~Practice~~**
- ~~Take a sample application and outline how it would be containerized.~~

---

### ~~Docker~~
**~~Theory~~**
- ~~Review Docker fundamentals and core commands.~~
- ~~Study official documentation on images, containers, volumes, and networks.~~

**~~Practice~~**
- ~~Write a simple Dockerfile for a “Hello World” application.~~
- ~~Build the image and run a container locally.~~

---

### ~~Kubernetes~~
**~~Theory~~**
- ~~Review core concepts: Pods, Services, Deployments, ConfigMaps, StatefulSets.~~
- ~~Review Kubernetes architecture and components.~~

**Practice**
- ~~Deploy a basic application in a local Kubernetes environment.~~
- ~~Practice common `kubectl` commands.~~

---

### ~~Helm~~
**~~Theory~~**
- ~~Review Helm concepts, chart structure, and templating~~.

**Practice**
- ~~Install an application using an existing Helm chart.~~
- ~~Modify values and perform an upgrade.~~

---

### ~~Bash Scripting~~
**~~Theory~~**
- ~~Review Bash syntax and scripting fundamentals~~.

**Practice**
- ~~Write and execute a short Bash script to automate a simple task.~~

---

### ~~Python Scripting~~
**~~Theory~~**
- ~~Review Python basics relevant to automation~~.

**~~Practic~~e**
- ~~Write and run a small Python script for a DevOps-related task.~~

---

### ~~Markdown~~
**~~Theor~~y**
- ~~Review Markdown syntax for documentation.~~

**~~Practice~~**
- ~~Create a short Markdown document using headings, lists, links, and code blocks.~~

---

### MkDocs
**Theory**
- Learn how MkDocs generates static documentation from Markdown.

**Practice**
- Create a small MkDocs project and serve or build the site.

---

### ~~DevOps Principles~~
**~~Theory~~**
- ~~Review core DevOps principles: collaboration, automation, IaC, observability, continuous improvement.~~

**Practice**
- Analyze a past project and document how DevOps principles were applied.

---

### ~~CI/CD~~
**~~Theory~~**
- ~~Review CI/CD pipeline concepts and stages.~~

**~~Practice~~**
- ~~Analyze an existing CI/CD pipeline configuration step by step.~~

---

### ~~GitHub Usage~~
**~~Theory~~**
- ~~Review Git fundamentals, branching, pull requests, and repository management.~~

**~~Practice~~**
- ~~Create a repository.~~
- ~~Clone it, create a branch, commit changes, and open a pull request.~~

---

### GitOps Workflow
**Theory**
- Review GitOps principles and declarative configuration workflows.

**Practice**
- Describe a GitOps pipeline for a Kubernetes application.

---

### ~~Argo CD~~
**~~Theory~~**
- ~~Review Argo CD concepts: Applications, ApplicationSets, and sync behavior.~~

**Practice**
- Deploy a simple application using Argo CD.

---

### ~~Infrastructure as Code Overview~~
**~~Theory~~**
- ~~Review IaC concepts and tool categories.~~

**Practice**
- Automate a simple infrastructure component using IaC.

---

### ~~Ansible~~
**~~Theory~~**
- ~~Review inventories, playbooks, roles, and task structure.~~

**Practice**
- Write and run a basic Ansible playbook.

---

### ~~Terraform~~
**~~Theory~~**
- ~~Review providers, resources, state, and workflow.~~

**~~Practice~~**
- ~~Write a simple Terraform configuration and run `terraform plan`.~~

---

### LMA Fundamentals
**Theory**
- ~~Review logging, monitoring, and alerting fundamentals.~~

**Practice**
- Describe the LMA stack in use and inspect a monitoring instance.

---

### Prometheus
**Theory**
- ~~Review scraping, targets, PromQL, and recording rules.~~

**Practice**
- Execute sample PromQL queries against a running Prometheus instance.

---

### MKE
**Theory**
- ~~Study MKE architecture and integration with Mirantis tools.~~

**Practice**
- Deploy and manage a test cluster using MKE.

---

## 2. Foundational Skills — New Knowledge

### Flux CD
**Theory**
- Learn Flux CD architecture and GitOps implementation.

**Practice**
- Bootstrap Flux on a cluster and deploy resources from Git.

---

### Sveltos
**Theory**
- ~~Learn how Sveltos manages add-ons across clusters.~~

**Practice**
- `Deploy Sveltos and apply an add-on using a ClusterProfile.`

---

### VictoriaMetrics
**Theory**
- ~~Learn VictoriaMetrics architecture and use cases.~~

**Practice**
- Deploy VictoriaMetrics and configure Prometheus remote storage.

---

### OpenSearch
**Theory**
- `Learn OpenSearch components for log indexing and visualization.`

**Practice**
- Deploy OpenSearch and forward Kubernetes logs into it.

---

### Go Programming
**Theory**
- Learn Go syntax, types, structs, pointers, and concurrency model.

**Practice**
- Write a Go program that interacts with the Kubernetes API.

---

### Kubebuilder
**Theory**
- Learn how Kubebuilder scaffolds Kubernetes operators.

**Practice**
- Create a basic Kubernetes operator and implement reconcile logic.

---

## 3. k0rdent Fundamentals and Architecture

### k0rdent Overview
**Theory**
- ~~Study k0rdent purpose, scope, and core features.~~

**Practice**
- ~~Summarize k0rdent components and capabilities.~~

---

### k0rdent Architecture
**Theory**
- `Study management cluster design, controllers, and CRDs.`

**Practice**
- `Create an architecture diagram and document component roles.`

---

### Kubernetes Cluster Manager
**Theory**
- `Learn KCM lifecycle management using Cluster API.`

**Practice**
- `Deploy a child cluster and observe KCM resources and logs.`

---

### k0rdent State Manager
**Theory**
- ~~Learn how KSM enforces service state across clusters.~~

**Practice**
- ~~Deploy a service using a ServiceTemplate.~~

---

### Observability and FinOps
**Theory**
- Study the KOF component and its responsibilities.

**Practice**
- Verify observability components on a deployed cluster.

---

### k0rdent UI
**Theory**
- Learn UI capabilities and supported workflows.

**Practice**
- Deploy and use the k0rdent UI.

---

### Child Cluster Deployment
**Theory**
- ~~Learn supported deployment targets and configuration options.~~

**Practice**
- `Deploy clusters on Docker, AWS, and OpenStack.`

---

## 4. k0rdent Custom Resources Deep Dive

For each CRD:
- **Theory**: Understand purpose and structure.
- **Practice**: Create, inspect, or modify the resource.

Covered CRDs:
- Management
- `Release`
- ManagementBackup
- ~~ProviderTemplate~~
- ~~ProviderInterface~~
- ~~Credential~~
- ~~AccessManagement~~
- StateManagementProvider
- ~~ServiceTemplate~~
- ~~ServiceTemplateChain~~
- ~~MultiClusterService~~
- ~~ClusterTemplate~~
- ~~ClusterTemplateChain~~
- ~~ClusterDeployment~~
- Cluster IPAM
- Cluster IPAM Claim
- `Machine`
- `ClusterProfile`
- `Profile`
- ClusterConfig

---

## 5. Troubleshooting

### General Troubleshooting
**Theory**
- `Review structured problem-solving techniques.`

**Practice**
- Define a troubleshooting plan for a generic failure.

---

### Kubernetes Troubleshooting
**Theory**
- Study common Kubernetes failure modes and component logs.

**Practice**
- Simulate and resolve Kubernetes failures.

---

### k0rdent Troubleshooting
**Theory**
- `Study k0rdent-specific failure scenarios.`

**Practice**
- Induce controlled k0rdent failures and recover from them.

---

## 6. Labs and Integrated Practice

### Lab Environment Setup
**Theory**
- `Define requirements for a multi-cluster lab environment`.

**Practice**
- `Finalize lab setup, tooling, access, and backups.`

---

### Hands-On Practice Project
**Theory**
- Plan an end-to-end multi-cluster project.

**Practice**
- `Deploy clusters across environments.`
- `Implement GitOps.`
- Apply LMA.
- Test backup and restore.
- `Simulate failures and perform troubleshooting.`
