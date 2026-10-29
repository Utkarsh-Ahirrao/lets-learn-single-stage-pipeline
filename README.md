# 🚀 Azure DevOps Single-Stage Terraform Pipeline

## 📌 Pipeline Overview

This Azure DevOps YAML pipeline automates the basic Terraform workflow. It connects Azure DevOps to Azure, checks the Terraform configuration, creates an execution plan, and applies the planned changes to the Azure infrastructure.

The pipeline contains **one stage: `Terraform`**, with one job and four tasks.

### 🔄 Pipeline Workflow

```text
Code pushed to master
        ⬇️
Azure DevOps Pipeline Trigger
        ⬇️
Self-Hosted Agent (Jarvis)
        ⬇️
Terraform Init
        ⬇️
Terraform Validate
        ⬇️
Terraform Plan
        ⬇️
Terraform Apply
        ⬇️
Azure Infrastructure Updated
```

> ⚠️ **Important:** The `apply` task can create, update, or delete infrastructure according to the Terraform plan. For production environments, consider adding approval checks before applying changes.

---

## 🧩 1. Pipeline Trigger

```yaml
trigger:
- master
```

This tells Azure DevOps to automatically start the pipeline when changes are pushed to the `master` branch.

**Why is it needed?**

Without a trigger, the pipeline may need to be started manually. A branch trigger helps automate the workflow whenever new code is pushed to the selected branch.

📌 In this configuration, the trigger applies to `master`.

---

## 🖥️ 2. Agent Pool

```yaml
pool: Jarvis
```

An **agent** is a machine that runs the pipeline tasks.

Here, `Jarvis` is the name of the agent pool configured for this pipeline. It is expected to provide the machine that executes the Terraform commands.

**Why is it needed?**

Azure DevOps needs a machine to download the repository, run Terraform, connect to Azure, and execute each task.

The agent must have the required tools and network access. If `Jarvis` is a self-hosted agent, Terraform and any required dependencies must be installed and configured on that machine.

---

## 🏗️ 3. Stage, Job, and Tasks

```yaml
stages:
  - stage: Terraform
    displayName: Terraform Validation and Plan
    jobs:
      - job: TerraformJob
```

These YAML sections organize the pipeline:

- **Stage:** A major section of the pipeline. Here, the pipeline has one stage called `Terraform`.
- **Job:** A group of tasks that run together on an agent.
- **Tasks:** Individual operations such as Terraform Init, Validate, Plan, and Apply.

Although the stage display name says *Validation and Plan*, the stage also contains an Apply task. A clearer display name would be **Terraform Init, Validate, Plan and Apply**.

---

## ⚙️ 4. Terraform Tasks Explained

### 🔹 Task 1: Terraform Init

```yaml
- task: TerraformTask@5
  displayName: Terraform Init
  inputs:
    provider: 'azurerm'
    command: 'init'
    workingDirectory: '$(System.DefaultWorkingDirectory)/Environments/Prod'
```

**What does it do?**

Terraform Init prepares the working directory before Terraform can perform other operations.

It initializes the Azure backend configuration, connects to the configured state storage, and downloads the required Terraform providers and modules.

**Why is it needed?**

Terraform needs its providers and backend initialized before it can validate the configuration or create a plan.

### ☁️ Remote Backend Configuration

```yaml
backendServiceArm: 'UtConnection'
backendAzureRmResourceGroupName: 'rg-backend-storage'
backendAzureRmStorageAccountName: 'backendstorage102026'
backendAzureRmContainerName: 'statefile-container'
backendAzureRmKey: 'utstate.tf'
```

These settings tell Terraform where to store its state file in Azure.

| Setting | Purpose |
|---|---|
| `backendServiceArm` | Uses the Azure DevOps service connection to authenticate for backend setup and access. |
| `backendAzureRmResourceGroupName` | Resource group containing the storage account. |
| `backendAzureRmStorageAccountName` | Storage account used for remote state. |
| `backendAzureRmContainerName` | Blob container that stores the state file. |
| `backendAzureRmKey` | Name of the state blob, here `utstate.tf`. |

**What is Terraform state?**

Terraform state records information about the infrastructure Terraform manages. It helps Terraform compare the current infrastructure with the configuration and identify what needs to change.

**Why use a remote backend?**

A remote backend keeps state in shared storage instead of only on a developer's local machine. This makes it easier for authorized team members and pipelines to work with the same state.

🔐 The state file may contain sensitive infrastructure information. Access to the storage account and container must be restricted.

---

### 🔹 Task 2: Terraform Validate

```yaml
- task: TerraformTask@5
  displayName: Terraform Validate
  inputs:
    provider: 'azurerm'
    command: 'validate'
    workingDirectory: '$(System.DefaultWorkingDirectory)/Environments/Prod'
    environmentServiceNameAzureRM: 'UtConnection'
```

**What does it do?**

Terraform Validate checks whether the Terraform configuration is structurally valid.

It can identify problems such as invalid arguments, incorrect block structures, and unsupported configuration settings.

**Why is it needed?**

It helps catch configuration mistakes before Terraform attempts to plan infrastructure changes.

📌 Validate does not prove that Azure resources can be created successfully. Permissions, quotas, existing resources, and other runtime issues can still cause failures later.

---

### 🔹 Task 3: Terraform Plan

```yaml
- task: TerraformTask@5
  displayName: Terraform Plan
  inputs:
    provider: 'azurerm'
    command: 'plan'
    workingDirectory: '$(System.DefaultWorkingDirectory)/Environments/Prod'
    environmentServiceNameAzureRM: 'UtConnection'
```

**What does it do?**

Terraform Plan compares the configuration with the state and the infrastructure information it reads from Azure. It calculates the changes Terraform proposes to make.

For example, the plan might show:

- ➕ Create a new virtual machine.
- 🔄 Update a network security rule.
- ➖ Delete a resource that Terraform is configured to remove.

**Why is it needed?**

The plan helps us understand the expected impact before making infrastructure changes.

📌 **Plan does not normally apply the changes.** It previews the proposed operations.

---

### 🔹 Task 4: Terraform Apply

```yaml
- task: TerraformTask@5
  inputs:
    provider: 'azurerm'
    command: 'apply'
    workingDirectory: '$(System.DefaultWorkingDirectory)/Environments/Prod'
    environmentServiceNameAzureRM: 'UtConnection'
```

**What does it do?**

Terraform Apply executes the infrastructure changes required by the configuration.

For example, it may create a VM, update an NSG rule, or remove a resource.

**Why is it needed?**

Plan tells us what Terraform proposes to change. Apply performs those changes.

⚠️ In this YAML, Apply runs automatically as part of the same job after Plan succeeds. The pipeline does not show a separate manual approval step.

For safer production deployments, consider separating planning and applying, reviewing the plan, and adding an approval gate where appropriate.

---

# 🔐 5. Azure Service Connection

## What is a Service Connection?

An Azure DevOps service connection is a configured connection that allows a pipeline to authenticate with Azure.

In this project, the service connection is named:

```yaml
UtConnection
```

The Terraform tasks use this connection to access Azure for operations that require authentication.

### 🤔 Why do we need it?

Imagine Azure DevOps is a person who wants to manage resources in your Azure subscription.

Azure needs to verify two things:

1. **Who is making the request?**
2. **What is that identity allowed to do?**

The service connection provides the authentication setup. Azure role assignments determine what the connected identity can do.

Without valid authentication and sufficient permissions, Terraform cannot perform the required Azure operations.

### 🔗 How does the connection work?

```text
Azure DevOps Pipeline
        ⬇️
UtConnection
        ⬇️
Configured Azure Identity
        ⬇️
Microsoft Entra ID Authentication
        ⬇️
Azure Role Assignments Checked
        ⬇️
Access to Permitted Azure Resources
```

The exact authentication method depends on how the service connection was configured. It may use workload identity federation or a service principal with credentials.

---

# 🪪 6. App Registration and Service Principal

## What is an App Registration?

An App Registration in Microsoft Entra ID defines an application identity that can be used for authentication.

In a typical service-principal-based setup, the app registration is associated with a **service principal** in the Azure tenant. The service principal is the identity that receives permissions to access Azure resources.

### Why is it needed?

A pipeline needs an identity to authenticate to Azure. Instead of using a person's everyday account, the pipeline can use a dedicated application identity.

This makes automation more manageable and helps separate pipeline access from personal user access.

### App Registration vs. Service Connection

| Component | Simple explanation |
|---|---|
| App Registration | Defines the application identity in Microsoft Entra ID. |
| Service Principal | The identity representation used to access resources in a particular tenant. |
| Azure DevOps Service Connection | Stores the configuration Azure DevOps uses to authenticate as the configured identity. |
| Azure RBAC Role | Defines what the identity is allowed to do and at which scope. |

**Important:** Creating an app registration alone does not automatically grant access to Azure resources. The identity also needs the appropriate authentication configuration and permissions.

---

# 🛡️ 7. Why Does the Pipeline Need Contributor Permissions?

Azure uses Role-Based Access Control (RBAC) to decide which identities can perform which operations.

For example, Terraform may need permission to create a virtual machine, configure a network interface, or update a network security group.

A commonly used role for these infrastructure operations is **Contributor**.

### What does Contributor allow?

Contributor generally allows an identity to manage Azure resources within its assigned scope, but it does not allow that identity to assign Azure RBAC roles to others.

For example, if the service principal has Contributor at the `rg-prod` resource group scope, it can generally manage supported resources in that resource group, subject to other restrictions.

### Where should it be assigned?

Assign the role to the service principal associated with `UtConnection` at the narrowest scope that supports the required operations.

For example:

```text
Microsoft Entra ID
       ⬇️
Service Principal used by UtConnection
       ⬇️
Azure Subscription / Resource Group
       ⬇️
Contributor Role Assignment
       ⬇️
Terraform can manage permitted infrastructure
```

📌 Contributor is a role assigned to the **service principal at an Azure resource scope**. It is not a permission that you simply switch on inside the app registration itself.

🔐 Follow least privilege: avoid granting Subscription-level Contributor when a resource-group-level assignment is sufficient.

---

# 💾 8. Why Does the Terraform Backend Need Permissions?

The backend is different from the infrastructure Terraform creates.

In this project:

- `rg-prod` contains the infrastructure being managed.
- `rg-backend-storage` contains the storage account used to keep Terraform state.
- The state blob is stored in `statefile-container` under the name `utstate.tf`.

Terraform needs access to the backend before it can reliably read and update the state used to manage infrastructure.

### What permissions are required?

The required permissions depend on how the AzureRM backend is configured to authenticate.

**For Azure Blob data access:** the identity generally needs the **Storage Blob Data Contributor** role on the appropriate storage account or container when using Microsoft Entra ID authentication for state access. This permits the required blob data operations, subject to scope and configuration.

**For storage account key access:** if the backend setup obtains a storage account key through Azure Resource Manager, the identity also needs permission to list or retrieve that key. A suitable management-plane role or custom role must grant the required operation. Contributor at the appropriate storage account scope commonly provides this capability.

These are different permissions:

- **Contributor** manages Azure resources through the management plane.
- **Storage Blob Data Contributor** accesses blob data through the data plane.

One does not automatically replace the other.

### Why might Contributor be used for the backend?

Some backend setups use the service connection to retrieve the storage account key during initialization. Contributor at the relevant storage account scope can provide the management-plane permission needed for that key retrieval.

However, assigning Contributor does not, by itself, guarantee direct blob data access through Microsoft Entra ID. The exact permissions must match the backend's authentication method.

### Recommended permission checks

Before running the pipeline, confirm:

- ✅ `UtConnection` authenticates successfully.
- ✅ Its service principal has the necessary permissions to manage the target infrastructure.
- ✅ The backend identity can access the storage account and the state blob.
- ✅ Any required storage-key retrieval permission is available if the configuration uses a storage key.
- ✅ The state container and blob are accessible to the pipeline.

**Security tip:** Grant permissions only at the scope required for the task. Protect Terraform state because it can contain sensitive values.

---

# 📂 9. Working Directory

The Terraform tasks use:

```yaml
workingDirectory: '$(System.DefaultWorkingDirectory)/Environments/Prod'
```

`$(System.DefaultWorkingDirectory)` points to the checked-out repository directory on the agent.

The pipeline then runs Terraform from `Environments/Prod`, where the production environment's Terraform configuration is expected to be located.

**Why is it needed?**

A repository can contain multiple environments or modules. The working directory tells Terraform which configuration to use for these tasks.

If this path is incorrect or the expected Terraform files are missing, the pipeline can fail.

---

# 📊 10. How to Read Pipeline Logs

When a pipeline runs, each task displays its status and logs.

| Task | What to check |
|---|---|
| Terraform Init | Backend authentication, state access, provider downloads and initialization. |
| Terraform Validate | Configuration errors and validation results. |
| Terraform Plan | Proposed creates, updates and deletes; check for unexpected changes. |
| Terraform Apply | Whether the planned infrastructure operations completed successfully. |

### Common issues

- 🔑 **Authentication error:** Check the service connection and its identity configuration.
- 🚫 **Authorization error:** Check the relevant Azure role assignments and scopes.
- 💾 **Backend access error:** Check storage access, blob permissions, and key retrieval permissions if applicable.
- 📁 **Directory error:** Check the working directory and the location of the Terraform files.
- ⚠️ **Resource conflict:** Check whether the resource already exists and whether Terraform state tracks it.

A green pipeline indicates that the configured tasks completed successfully. It does not replace a review of the plan or verification that the resulting infrastructure meets the intended requirements.

---

## 🎯 Final Summary

This learning repository demonstrates how a basic Azure DevOps pipeline can automate Terraform operations.

- 🚦 **Trigger** starts the pipeline when code is pushed to `master`.
- 🖥️ **Agent** provides the machine that runs the tasks.
- ⚙️ **Init** prepares Terraform and its remote backend.
- ✅ **Validate** checks the configuration.
- 🔍 **Plan** previews the proposed infrastructure changes.
- 🚀 **Apply** executes the changes.
- 🔐 **Service Connection and App Registration/Service Principal** provide the identity and authentication setup.
- 🛡️ **Azure RBAC** controls which resources the pipeline can access and manage.
- 💾 **Backend permissions** allow Terraform to read and update the remote state.

The goal is to understand not only how to write the YAML, but also why the pipeline needs an agent, authentication, permissions, and remote state storage.
