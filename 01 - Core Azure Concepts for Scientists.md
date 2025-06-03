# **Azure Cloud Primer for Scientists: Migrating from AWS**

This primer is designed for scientists and research teams familiar with Amazon Web Services (AWS) who are migrating workloads and repositories to Microsoft Azure. It focuses on translating key AWS concepts and services to their Azure equivalents, with particular attention to Azure Blob Storage, Azure AI Foundry, and the use of Shared Access Signature (SAS) tokens. The goal is to facilitate a smoother transition by providing clear explanations, term mappings, and practical guidance for leveraging Azure's capabilities in scientific computing and data-intensive research.

## **1\. Core Azure Concepts for Scientists: Mapping from AWS**

Transitioning from AWS to Azure involves understanding a new set of terminologies and organizational structures for managing cloud resources. This section provides a high-level mapping of core concepts.

### **1.1. Resource Organization in Azure vs. AWS**

In AWS, resources are typically organized within AWS Accounts, which can be grouped under AWS Organizations using Organizational Units (OUs) for policy management and billing consolidation.

Azure employs a slightly different, albeit conceptually similar, hierarchical structure:

* **Azure Subscription:** An Azure subscription is a billing and management boundary. It is associated with a Microsoft Entra ID tenant (discussed later) and provides access to Azure services. This is somewhat analogous to an AWS Account in terms of being a primary container for resources and billing.  
* **Resource Group:** A Resource Group is a container that holds related resources for an Azure solution. Think of it as a logical grouping for all the components of a project or application (e.g., virtual machines, storage accounts, databases). All resources within a resource group must reside in the same Azure region, but a resource group can contain resources from different services. Deleting a resource group deletes all resources within it. This concept doesn't have a direct one-to-one equivalent in AWS but is fundamental to Azure resource management, offering a more granular organization within a subscription than simply tagging resources in an AWS account.  
* **Management Groups:** For organizations with multiple subscriptions, Azure Management Groups provide a level of scope above subscriptions. Policies and access controls can be applied at the management group level, and these are inherited by underlying subscriptions and resource groups. This is comparable to AWS Organizations and OUs for hierarchical governance. 1

Understanding this hierarchy is crucial for organizing projects, managing access, and controlling costs effectively in Azure.

### **1.2. Compute Services: Azure Equivalents for AWS EC2, Batch, and Lambda**

Scientific workloads often rely heavily on various compute services. Here’s how common AWS compute offerings map to Azure:

* **Virtual Machines (Azure Virtual Machines vs. AWS EC2):**  
  * AWS's Amazon Elastic Compute Cloud (EC2) instances find their direct counterpart in **Azure Virtual Machines (VMs)**. Both platforms offer a wide array of VM sizes (instance types in AWS) optimized for different workloads (general purpose, compute-optimized, memory-optimized, GPU-enabled, etc.). 2 Both AWS and Azure bill on-demand VMs per second used. While categories are similar, exact RAM, CPU, and storage capabilities can differ, so careful selection based on workload requirements is important. 2 Azure also provides **Data Science Virtual Machines (DSVMs)**, which are pre-configured VM images with popular data science tools, libraries (Python, R, Julia), and frameworks (TensorFlow, PyTorch), accelerating setup for AI and ML tasks. 3  
* **Autoscaling (Azure Virtual Machine Scale Sets & App Service Autoscale vs. AWS Auto Scaling):**  
  * AWS Auto Scaling allows automatic adjustment of EC2 instance capacity. In Azure, **Virtual Machine Scale Sets** enable the deployment and management of identical VMs, with the number of instances capable of autoscaling based on demand or a schedule. 2 For web applications and APIs hosted on Azure App Service, **App Service Autoscale** provides similar functionality. 2  
* **Batch Processing (Azure Batch vs. AWS Batch):**  
  * For large-scale parallel and high-performance computing (HPC) workloads, AWS Batch is a common choice. Azure offers **Azure Batch**, a service that helps manage compute-intensive work across a scalable collection of VMs. 2 Azure Batch allows scientists to schedule, manage, and monitor batch jobs, dynamically provisioning resources as needed. It integrates with Azure Storage for data staging and retrieval.  
* **Serverless Computing (Azure Functions vs. AWS Lambda):**  
  * AWS Lambda provides serverless, event-driven compute. The primary Azure equivalent is **Azure Functions**. 2 Both services allow running code in response to various triggers (e.g., HTTP requests, new data in storage, messages in a queue) without managing underlying server infrastructure. Azure Functions supports multiple languages, including Python, Node.js, Java, and C\#.

The following table summarizes the compute service mapping:

| AWS Service | Azure Service | Description |
| :---- | :---- | :---- |
| Amazon EC2 Instance Types | Azure Virtual Machines | On-demand VMs with various configurations for different workloads. 2 |
| VMware Cloud on AWS | Azure VMware Solution | Move VMware vSphere-based workloads to the cloud. 2 |
| AWS Parallel Cluster | Azure CycleCloud | Create, manage, operate, and optimize HPC and large compute clusters. 2 |
| AWS Auto Scaling | Virtual Machine Scale Sets, App Service Autoscale | Automatically adjust the number of VM instances or app service instances based on demand. 2 |
| AWS Batch | Azure Batch | Manage compute-intensive work across a scalable collection of VMs for batch processing. 2 |
| AWS Lambda | Azure Functions, WebJobs in Azure App Service | Serverless, event-driven code execution. WebJobs offer scheduled or continuous background task execution within App Service. 2 |

### **1.3. Database Services for Scientific Data**

Scientific research generates and consumes diverse datasets, requiring various database solutions.

* **Relational Databases (Azure SQL Database family vs. AWS RDS):**  
  * Amazon Relational Database Service (Amazon RDS) supports various database engines. Azure provides several managed relational database services, including **Azure SQL Database** (for SQL Server), **Azure Database for MySQL**, and **Azure Database for PostgreSQL**. 6 These services handle much of the underlying infrastructure management, patching, and backups. For workloads requiring OS-level access or specific configurations not available in PaaS offerings, database engines like SQL Server, Oracle, or MySQL can also be deployed on Azure VMs. 6  
  * **Serverless Relational:** AWS offers Aurora Serverless. Azure provides **Azure SQL Database serverless** and **Serverless SQL pools in Azure Synapse Analytics**, which automatically scale compute based on workload demand and bill for actual compute used or data processed. 6  
* **NoSQL Databases (Azure Cosmos DB vs. AWS DynamoDB, DocumentDB, Neptune):**  
  * AWS provides several NoSQL options like DynamoDB (key-value and document), Amazon DocumentDB (MongoDB-compatible), and Amazon Neptune (graph). Azure's flagship NoSQL service is **Azure Cosmos DB**. 6 Azure Cosmos DB is a globally distributed, multi-model database service supporting key-value, document, column-family, graph, and vector data models. This flexibility makes it suitable for a wide range of scientific applications with varying data structures and access patterns.  
* **Data Warehousing and Analytics (Azure Synapse Analytics vs. AWS Redshift):**  
  * AWS Redshift is a petabyte-scale data warehousing service. Azure's comprehensive solution in this space is **Azure Synapse Analytics**. 5 Azure Synapse Analytics is a unified analytics platform that combines enterprise data warehousing, big data processing (using Apache Spark), data integration pipelines, and serverless query capabilities for data in data lakes. This integrated approach can simplify complex analytical workflows.

The table below offers a comparative view:

| Type | AWS Service | Azure Service | Description |
| :---- | :---- | :---- | :---- |
| Relational Database | Amazon RDS | Azure SQL Database, Azure Database for MySQL, Azure Database for PostgreSQL | Managed relational database services. 6 |
| Serverless Relational | Amazon Aurora Serverless | Azure SQL Database serverless, Serverless SQL pool in Azure Synapse Analytics | Relational databases that automatically scale compute based on workload demand. 6 |
| NoSQL | Amazon DynamoDB (Key-Value/Document), Amazon DocumentDB, Amazon Neptune (Graph) | Azure Cosmos DB | Azure Cosmos DB is a globally distributed, multi-model database supporting key-value, document, graph, and columnar models. 6 |
| Data Warehousing/Analytics | Amazon Redshift | Azure Synapse Analytics | Petabyte-scale data warehousing and unified analytics platforms. Azure Synapse Analytics integrates data warehousing, big data processing, and data integration. 5 |
| Caching | Amazon ElastiCache, Amazon MemoryDB for Redis | Azure Cache for Redis | In-memory distributed caching services. 6 |
| Database Migration | AWS Database Migration Service | Azure Database Migration Service | Services to facilitate migration of database schemas and data to the cloud. 6 |

### **1.4. Identity and Access Management (Microsoft Entra ID & RBAC vs. AWS IAM)**

Securing access to cloud resources is paramount.

* AWS Identity and Access Management (IAM) is used to manage access to AWS services and resources securely. It allows the creation and management of users, groups, roles, and policies.  
* In Azure, identity and access management is primarily handled by **Microsoft Entra ID (formerly Azure Active Directory)** and **Azure Role-Based Access Control (RBAC)**. 1  
  * **Microsoft Entra ID:** This is Azure's cloud-based identity and access management service. It manages users, groups, and application identities (service principals, managed identities). It provides authentication (verifying who a user or service is) and can be synchronized with on-premises Active Directory. 8 It also supports Single Sign-On (SSO) and Multi-Factor Authentication (MFA). 1  
  * **Azure RBAC:** This system is used for authorization (determining what an authenticated user or service can do). Azure RBAC allows granting specific permissions (e.g., read, write, delete) to users, groups, service principals, or managed identities on Azure resources, resource groups, or subscriptions. Permissions are defined through roles (e.g., Reader, Contributor, Owner, or custom roles). 1

Essentially, Microsoft Entra ID handles *who* can access, while Azure RBAC determines *what* they can do with specific Azure resources. This is conceptually similar to how AWS IAM users/roles are granted permissions via IAM policies.

| AWS Service | Azure Service | Description |
| :---- | :---- | :---- |
| AWS Identity and Access Management (IAM) | Microsoft Entra ID | Centralized identity management service for users, groups, and applications; provides authentication, SSO, MFA. 1 |
| AWS IAM Policies / Roles | Azure Role-Based Access Control (RBAC) | Manages authorization by assigning roles (collections of permissions) to identities for specific Azure resources, resource groups, or subscriptions. 1 |
| AWS Organizations | Azure Management Groups | Hierarchical organization structure to manage multiple accounts/subscriptions with inherited policies. 1 |
| AWS IAM Identity Center (formerly AWS SSO) | Microsoft Entra ID SSO | Centralized access management enabling users to access multiple applications with a single set of credentials. 1 |
| AWS Directory Service | Microsoft Entra Domain Services | Managed directory services providing domain join, Group Policy, LDAP, and Kerberos/NTLM authentication for compatibility with traditional on-premises AD. 1 |
