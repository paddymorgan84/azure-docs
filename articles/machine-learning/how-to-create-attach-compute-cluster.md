---
title: Create compute clusters
titleSuffix: Azure Machine Learning
description: Learn how to create compute clusters in your Azure Machine Learning workspace. Use the compute cluster as a compute target for training or inference.
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: how-to
ms.custom: devx-track-azurecli, cliv2, sdkv1, event-tier1-build-2022
ms.author: sgilley
author: sdgilley
ms.reviewer: sgilley
ms.date: 08/05/2022
---

# Create an Azure Machine Learning compute cluster

[!INCLUDE [sdk v1](../../includes/machine-learning-sdk-v1.md)]
[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]

> [!div class="op_single_selector" title1="Select the Azure Machine Learning CLI version you are using:"]
> * [CLI v1](v1/how-to-create-attach-compute-cluster.md)
> * [CLI v2 (current version)](how-to-create-attach-compute-cluster.md)

Learn how to create and manage a [compute cluster](concept-compute-target.md#azure-machine-learning-compute-managed) in your Azure Machine Learning workspace.

You can use Azure Machine Learning compute cluster to distribute a training or batch inference process across a cluster of CPU or GPU compute nodes in the cloud. For more information on the VM sizes that include GPUs, see [GPU-optimized virtual machine sizes](../virtual-machines/sizes-gpu.md). 

In this article, learn how to:

* Create a compute cluster
* Lower your compute cluster cost
* Set up a [managed identity](../active-directory/managed-identities-azure-resources/overview.md) for the cluster

## Prerequisites

* An Azure Machine Learning workspace. For more information, see [Create an Azure Machine Learning workspace](how-to-manage-workspace.md).

* The [Azure CLI extension for Machine Learning service (v2)](reference-azure-machine-learning-cli.md), [Azure Machine Learning Python SDK](/python/api/overview/azure/ml/intro), or the [Azure Machine Learning Visual Studio Code extension](how-to-setup-vs-code.md).

* If using the Python SDK, [set up your development environment with a workspace](how-to-configure-environment.md).  Once your environment is set up, attach to the workspace in your Python script:

    [!INCLUDE [sdk v1](../../includes/machine-learning-sdk-v1.md)]

    ```python
    from azureml.core import Workspace
    
    ws = Workspace.from_config() 
    ```

## What is a compute cluster?

Azure Machine Learning compute cluster is a managed-compute infrastructure that allows you to easily create a single or multi-node compute. The compute cluster is a resource that can be shared with other users in your workspace. The compute scales up automatically when a job is submitted, and can be put in an Azure Virtual Network. Compute cluster supports **no public IP (preview)** deployment as well in virtual network. The compute executes in a containerized environment and packages your model dependencies in a [Docker container](https://www.docker.com/why-docker).

Compute clusters can run jobs securely in a [virtual network environment](how-to-secure-training-vnet.md), without requiring enterprises to open up SSH ports. The job executes in a containerized environment and packages your model dependencies in a Docker container. 

## Limitations

* Some of the scenarios listed in this document are marked as __preview__. Preview functionality is provided without a service level agreement, and it's not recommended for production workloads. Certain features might not be supported or might have constrained capabilities. For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

* Compute clusters can be created in a different region than your workspace. This functionality is in __preview__, and is only available for __compute clusters__, not compute instances. This preview isn't available if you're using a private endpoint-enabled workspace. 

    > [!WARNING]
    > When using a compute cluster in a different region than your workspace or datastores, you may see increased network latency and data transfer costs. The latency and costs can occur when creating the cluster, and when running jobs on it.

* We currently support only creation (and not updating) of clusters through [ARM templates](/azure/templates/microsoft.machinelearningservices/workspaces/computes). For updating compute, we recommend using the SDK, Azure CLI or UX for now.

* Azure Machine Learning Compute has default limits, such as the number of cores that can be allocated. For more information, see [Manage and request quotas for Azure resources](how-to-manage-quotas.md).

* Azure allows you to place _locks_ on resources, so that they can't be deleted or are read only. __Do not apply resource locks to the resource group that contains your workspace__. Applying a lock to the resource group that contains your workspace will prevent scaling operations for Azure ML compute clusters. For more information on locking resources, see [Lock resources to prevent unexpected changes](../azure-resource-manager/management/lock-resources.md).

> [!TIP]
> Clusters can generally scale up to 100 nodes as long as you have enough quota for the number of cores required. By default clusters are setup with inter-node communication enabled between the nodes of the cluster to support MPI jobs for example. However you can scale your clusters to 1000s of nodes by simply [raising a support ticket](https://portal.azure.com/#blade/Microsoft_Azure_Support/HelpAndSupportBlade/newsupportrequest), and requesting to allow list your subscription, or workspace, or a specific cluster for disabling inter-node communication.

## Create

**Time estimate**: Approximately 5 minutes.

Azure Machine Learning Compute can be reused across runs. The compute can be shared with other users in the workspace and is retained between runs, automatically scaling nodes up or down based on the number of runs submitted, and the max_nodes set on your cluster. The min_nodes setting controls the minimum nodes available.

The dedicated cores per region per VM family quota and total regional quota, which applies to compute cluster creation, is unified and shared with Azure Machine Learning training compute instance quota. 

[!INCLUDE [min-nodes-note](../../includes/machine-learning-min-nodes.md)]

The compute autoscales down to zero nodes when it isn't used.   Dedicated VMs are created to run your jobs as needed.
    
# [Python](#tab/python)

To create a persistent Azure Machine Learning Compute resource in Python, specify the **vm_size** and **max_nodes** properties. Azure Machine Learning then uses smart defaults for the other properties.
    
* **vm_size**: The VM family of the nodes created by Azure Machine Learning Compute.
* **max_nodes**: The max number of nodes to autoscale up to when you run a job on Azure Machine Learning Compute.

[!INCLUDE [sdk v1](../../includes/machine-learning-sdk-v1.md)]

[!code-python[](~/aml-sdk-samples/ignore/doc-qa/how-to-set-up-training-targets/amlcompute2.py?name=cpu_cluster)]

You can also configure several advanced properties when you create Azure Machine Learning Compute. The properties allow you to create a persistent cluster of fixed size, or within an existing Azure Virtual Network in your subscription.  See the [AmlCompute class](/python/api/azureml-core/azureml.core.compute.amlcompute.amlcompute) for details.

> [!WARNING]
> When setting the `location` parameter, if it is a different region than your workspace or datastores you may see increased network latency and data transfer costs. The latency and costs can occur when creating the cluster, and when running jobs on it.

# [Azure CLI](#tab/azure-cli)

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]

```azurecli
az ml compute create -f create-cluster.yml
```

Where the file *create-cluster.yml* is:

:::code language="yaml" source="~/azureml-examples-main/cli/resources/compute/cluster-location.yml":::


> [!WARNING]
> When using a compute cluster in a different region than your workspace or datastores, you may see increased network latency and data transfer costs. The latency and costs can occur when creating the cluster, and when running jobs on it.


# [Studio](#tab/azure-studio)

Create a single- or multi- node compute cluster for your training, batch inferencing or reinforcement learning workloads. 

1. Navigate to [Azure Machine Learning studio](https://ml.azure.com).
 
1. Under __Manage__, select __Compute__.
1. If you have no compute resources, select  **Create** in the middle of the page.
  
    :::image type="content" source="media/how-to-create-attach-studio/create-compute-target.png" alt-text="Screenshot that shows creating a compute target":::

1. If you see a list of compute resources, select **+New** above the list.

    :::image type="content" source="media/how-to-create-attach-studio/select-new.png" alt-text="Select new":::

1. In the tabs at the top, select __Compute cluster__

1. Fill out the form as follows:

    |Field  |Description  |
    |---------|---------|
    | Location | The Azure region where the compute cluster will be created. By default, this is the same location as the workspace. Setting the location to a different region than the workspace is in __preview__, and is only available for __compute clusters__, not compute instances.</br>When using a different region than your workspace or datastores, you may see increased network latency and data transfer costs. The latency and costs can occur when creating the cluster, and when running jobs on it. |
    |Virtual machine type |  Choose CPU or GPU. This type can't be changed after creation     |
    |Virtual machine priority | Choose **Dedicated** or **Low priority**.  Low priority virtual machines are cheaper but don't guarantee the compute nodes. Your job may be preempted.
    |Virtual machine size     |  Supported virtual machine sizes might be restricted in your region. Check the [availability list](https://azure.microsoft.com/global-infrastructure/services/?products=virtual-machines)     |

1. Select **Next** to proceed to **Advanced Settings** and fill out the form as follows:

    |Field  |Description  |
    |---------|---------|
    |Compute name     | * Name is required and must be between 3 to 24 characters long.<br><br> * Valid characters are upper and lower case letters, digits, and the  **-** character.<br><br> * Name must start with a letter<br><br> * Name needs to be unique across all existing computes within an Azure region. You'll see an alert if the name you choose isn't unique<br><br> * If **-**  character is used, then it needs to be followed by at least one letter later in the name    |
    |Minimum number of nodes | Minimum number of nodes that you want to provision. If you want a dedicated number of nodes, set that count here. Save money by setting the minimum to 0, so you won't pay for any nodes when the cluster is idle. |
    |Maximum number of nodes | Maximum number of nodes that you want to provision. The compute will autoscale to a maximum of this node count when a job is submitted. |
    | Idle seconds before scale down | Idle time before scaling the cluster down to the minimum node count. |
    | Enable SSH access | Use the same instructions as [Enable SSH access](#enable-ssh-access) for a compute instance (above). |
    |Advanced settings     |  Optional. Configure a virtual network. Specify the **Resource group**, **Virtual network**, and **Subnet** to create the compute instance inside an Azure Virtual Network (vnet). For more information, see these [network requirements](./how-to-secure-training-vnet.md) for vnet.   Also attach [managed identities](#set-up-managed-identity) to grant access to resources.

1. Select __Create__.


### Enable SSH access

SSH access is disabled by default.  SSH access can't be changed after creation. Make sure to enable access if you plan to debug interactively with [VS Code Remote](how-to-set-up-vs-code-remote.md).  

[!INCLUDE [enable-ssh](../../includes/machine-learning-enable-ssh.md)]

### Connect with SSH access

[!INCLUDE [ssh-access](../../includes/machine-learning-ssh-access.md)]

---

 ## Lower your compute cluster cost

You may also choose to use [low-priority VMs](how-to-manage-optimize-cost.md#low-pri-vm) to run some or all of your workloads. These VMs don't have guaranteed availability and may be preempted while in use. You'll have to restart a preempted job. 

Use any of these ways to specify a low-priority VM:
    
# [Python](#tab/python)

[!INCLUDE [sdk v1](../../includes/machine-learning-sdk-v1.md)]

```python
compute_config = AmlCompute.provisioning_configuration(vm_size='STANDARD_D2_V2',
                                                            vm_priority='lowpriority',
                                                            max_nodes=4)
```
    
# [Azure CLI](#tab/azure-cli)

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]

Set the `vm-priority`:
    
```azurecli
az ml compute create -f create-cluster.yml
```

Where the file *create-cluster.yml* is:

:::code language="yaml" source="~/azureml-examples-main/cli/resources/compute/cluster-low-priority.yml":::

# [Studio](#tab/azure-studio)

In the studio, choose **Low Priority** when you create a VM.

--- 

## Set up managed identity

[!INCLUDE [aml-clone-in-azure-notebook](../../includes/aml-managed-identity-intro.md)]

# [Python](#tab/python)

[!INCLUDE [sdk v1](../../includes/machine-learning-sdk-v1.md)]

* Configure managed identity in your provisioning configuration:  

    * System assigned managed identity created in a workspace named `ws`
        ```python
        # configure cluster with a system-assigned managed identity
        compute_config = AmlCompute.provisioning_configuration(vm_size='STANDARD_D2_V2',
                                                                max_nodes=5,
                                                                identity_type="SystemAssigned",
                                                                )
        cpu_cluster_name = "cpu-cluster"
        cpu_cluster = ComputeTarget.create(ws, cpu_cluster_name, compute_config)
        ```
    
    * User-assigned managed identity created in a workspace named `ws`
    
        ```python
        # configure cluster with a user-assigned managed identity
        compute_config = AmlCompute.provisioning_configuration(vm_size='STANDARD_D2_V2',
                                                                max_nodes=5,
                                                                identity_type="UserAssigned",
                                                                identity_id=['/subscriptions/<subcription_id>/resourcegroups/<resource_group>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<user_assigned_identity>'])
    
        cpu_cluster_name = "cpu-cluster"
        cpu_cluster = ComputeTarget.create(ws, cpu_cluster_name, compute_config)
        ```

* Add managed identity to an existing compute cluster named `cpu_cluster`
    
    * System-assigned managed identity:
    
        ```python
        # add a system-assigned managed identity
        cpu_cluster.add_identity(identity_type="SystemAssigned")
        ````
    
    * User-assigned managed identity:
    
        ```python
        # add a user-assigned managed identity
        cpu_cluster.add_identity(identity_type="UserAssigned", 
                                    identity_id=['/subscriptions/<subcription_id>/resourcegroups/<resource_group>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/<user_assigned_identity>'])
        ```

# [Azure CLI](#tab/azure-cli)

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]


### Create a new managed compute cluster with managed identity

Use this command:

```azurecli
az ml compute create -f create-cluster.yml
```

Where the contents of *create-cluster.yml* are as follows: 

* User-assigned managed identity

    :::code language="yaml" source="~/azureml-examples-main/cli/resources/compute/cluster-user-identity.yml":::

* System-assigned managed identity

    :::code language="yaml" source="~/azureml-examples-main/cli/resources/compute/cluster-system-identity.yml":::

### Add a managed identity to an existing cluster

To update an existing cluster:

* User-assigned managed identity

    :::code language="azurecli" source="~/azureml-examples-main/cli/deploy-mlcompute-update-to-user-identity.sh":::

* System-assigned managed identity

    :::code language="azurecli" source="~/azureml-examples-main/cli/deploy-mlcompute-update-to-system-identity.sh":::


# [Studio](#tab/azure-studio)

During cluster creation or when editing compute cluster details, in the **Advanced settings**, toggle **Assign a managed identity** and specify a system-assigned identity or user-assigned identity.

---

[!INCLUDE [aml-clone-in-azure-notebook](../../includes/aml-managed-identity-note.md)]

### Managed identity usage

[!INCLUDE [aml-clone-in-azure-notebook](../../includes/aml-managed-identity-default.md)]

## Troubleshooting

There's a chance that some users who created their Azure Machine Learning workspace from the Azure portal before the GA release might not be able to create AmlCompute in that workspace. You can either raise a support request against the service or create a new workspace through the portal or the SDK to unblock yourself immediately.

### Stuck at resizing

If your Azure Machine Learning compute cluster appears stuck at resizing (0 -> 0) for the node state, this may be caused by Azure resource locks.

[!INCLUDE [resource locks](../../includes/machine-learning-resource-lock.md)]

## Next steps

Use your compute cluster to:

* [Submit a training run](how-to-set-up-training-targets.md) 
* [Run batch inference](./tutorial-pipeline-batch-scoring-classification.md).
