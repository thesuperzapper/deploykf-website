---
icon: custom/kubeflow
description: >-
  Learn how to configure Kubeflow Notebooks in deployKF.
  Use custom environments, GPU acceleration, faster storage, and more!
---
# Configure Kubeflow Notebooks

Learn how to __configure Kubeflow Notebooks__ in deployKF.

---

## Introduction

deployKF includes [Kubeflow Notebooks](../../reference/tools.md#kubeflow-notebooks) so users can run web-based development environments (IDEs) inside your Kubernetes cluster.
There is out-of-the-box support for [JupyterLab](https://jupyter.org/), [Visual Studio Code (code-server)](https://github.com/coder/code-server), and [RStudio](https://github.com/rstudio/rstudio).

### __About this Guide__

This guide will help you __configure Kubeflow Notebooks__ to suit your organization's needs.

The following [spawner options](#spawner-options) affect the "New Notebook" flow:

Spawner Option<br><small>(Click for Details)</small> | Description
--- | ---
[Container Images](#container-images) | Choose which IDE and packages are pre-installed in the Notebook.
[Container Resources](#container-resources) | Configure Pod resource [requests and limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), including CPU, Memory, and GPUs.
[Storage Volumes](#storage-volumes) | Mount [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) to the Notebook Pod for storing data between restarts.
[Affinity and Anti-Affinity](#affinity-and-anti-affinity) | Configure how Notebook Pods are [scheduled on nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) and [relative to other Pods](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity).
[Node Tolerations](#node-tolerations) | Configure tolerations which allow Pods to be scheduled on [tainted nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).
[PodDefaults](#poddefaults) | PodDefaults are a [Kubeflow CRD](https://github.com/kubeflow/kubeflow/tree/master/components/admission-webhook) to apply configs to Pods based on their labels.

The following [platform configs](#platform-configs) affect the behavior of the entire platform:

Platform Configs<br><small>(Click for Details)</small> | Description
--- | ---
[Dedicated Notebook Nodes](#dedicated-notebook-nodes) | Provision a dedicated node for each Notebook Pod.<br><br><small>:star: Consider this if you have a cluster-autoscaler and want better resource isolation. :star:</small>
[Idle Notebook Culling](#idle-notebook-culling) | Automatically cull idle Notebooks to save resources.
[Kubeflow Pipelines Authentication](#kubeflow-pipelines-authentication) | Easy authentication for Kubeflow Pipelines SDK in Notebooks.
[Override Notebook Template](#override-notebook-template) | Override the Notebook YAML template. (Advanced)

---

## __Spawner Options__

The spawner options are the configurations that are available to users when spawning a new Notebook.

!!! warning "Updating Spawner Options"

    When the `kubeflow_tools.notebooks.spawnerFormDefaults` values are updated, this __does NOT affect existing Notebooks__ as these configs are only used when __spawning__ a new Notebook.

!!! info "Kubeflow Notebooks 2.0"

    _Kubeflow Notebooks 1.0_ exposes many Kubernetes-specific concepts to users.
    There is upstream work on _Kubeflow Notebooks 2.0_ to abstract the Kubernetes configs in a better way, see [`kubeflow/kubeflow#7156`](https://github.com/kubeflow/kubeflow/issues/7156) and the [design document](https://docs.google.com/document/d/1_zk06zebbaTBdJ8TdU07Ibky25hqHGARXjVcsp2qEnU/edit) for more information.

### __Container Images__

Container images are the "environment" which users will be working in when using a Notebook Pod, and can be configured to provide different tools and packages to users.

When spawning a new Notebook, the user is presented the choice of three "groups" which each correspond to a different IDE.
Each group has a _list of container images_ which are defined by the cluster administrator using values.
Users choose an image by expanding the "Custom Notebook" dropdown, and selecting their desired image.

<div class="image-wrapper" markdown="1">
  ![Kubeflow Notebooks Spawner Groups (Dark Mode)](../../assets/images/kubeflow-notebooks-groups-DARK.png#only-dark)
  ![Kubeflow Notebooks Spawner Groups (Light Mode)](../../assets/images/kubeflow-notebooks-groups-LIGHT.png#only-light)
</div>

We provide a number of pre-built container images, but you may also customize the list of available images to suit your organization's needs.
The following values define the list of available container images in each group:

Group | Values
--- | ---
Jupyter-Like | [`kubeflow_tools.notebooks.spawnerFormDefaults.image`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1475-L1489)
VSCode-like (group 1) | [`kubeflow_tools.notebooks.spawnerFormDefaults.imageGroupOne`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1491-L1505)
RStudio-like (group 2) | [`kubeflow_tools.notebooks.spawnerFormDefaults.imageGroupTwo`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1507-L1523)

For example, to add a custom Jupyter-like image, you may use the following values:

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      
      ## Jupyter-like Container Images
      ##
      image:
        ## the default container image
        value: kubeflownotebookswg/jupyter-scipy:v1.7.0

        ## the list of available container images in the dropdown
        options:
          - kubeflownotebookswg/jupyter-scipy:v1.7.0
          - kubeflownotebookswg/jupyter-pytorch-full:v1.7.0
          - kubeflownotebookswg/jupyter-pytorch-cuda-full:v1.7.0
          - kubeflownotebookswg/jupyter-tensorflow-full:v1.7.0
          - kubeflownotebookswg/jupyter-tensorflow-cuda-full:v1.7.0
  
          ## your custom container image
          - <your-registry>/<your-image>:<your-tag>
```

!!! warning "User-Defined Custom Images"

    By default, users may provide a custom image string when spawning a Notebook, this could expose your cluster to significant security risks, even if you trust your users.

    Disallow user-provided images by setting [`kubeflow_tools.notebooks.spawnerFormDefaults.allowCustomImage`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1458-L1459) to `false`.
    Please note, this ONLY impacts the Notebook Spawner UI, and does NOT prevent users from manually creating Notebooks with custom images (if they have the necessary `kubectl` permissions).

    ```yaml
    kubeflow_tools:
      notebooks:
        spawnerFormDefaults:
          allowCustomImage: false
    ```

!!! info "Build a Custom Image"

    There are specific requirements to use an image with Kubeflow Notebooks, see the upstream guide on [building custom container images](https://www.kubeflow.org/docs/components/notebooks/container-images/#custom-images) for more information.

    Please note, other IDEs (without a dedicated group) may still work in Kubeflow Notebooks.
    In general, _group 1_ (the VSCode one) can be used with any IDE that serves relative HTTP links and meets the other container image requirements.

!!! info "Differences Between Groups"

    In addition to providing visual separation, there are slight behavioral differences between the groups to accommodate the different IDEs:

    - __Jupyter-like:__
        - The `NB_PREFIX` environment variable contains the HTTP path which the Notebook is being served under, typically `/notebook/{profile_name}/{notebook_name}/`.
    - __VSCode-like:__
        - The `NB_PREFIX` is the same as in "Jupyter-like".
        - Incoming HTTP requests have their paths rewritten to remove the `NB_PREFIX` prefix.
    - __RStudio-like:__
        - The `NB_PREFIX` is the same as in "Jupyter-like".
        - Incoming HTTP requests have their paths rewritten to remove the `NB_PREFIX` prefix.
        - Incoming HTTP requests have a `X-RStudio-Root-Path` header added, with the value of `NB_PREFIX`.

### __Container Resources__

Each Notebook runs inside a Kubernetes Pod, and can [request resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) including CPU, Memory, and GPUs like any other Pod.

It's important to understand that __resource requests__ primarily [determine how Pods are assigned to nodes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-requests-are-scheduled), 
whereas __resource limits__ typically [enforce a maximum quantity](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run) of a resource that a Pod can use before it is killed or throttled.

Resource | Request Behavior | Limit Exceeded Behavior
--- | --- | ---
CPU | Deciding where to schedule the Pod //<br> Allocating CPU time (when node is busy) | Processes CPU time is hard throttled
Memory | Deciding where to schedule the Pod //<br> [Memory QoS](https://kubernetes.io/blog/2023/05/05/qos-memory-resources/) (cgroups V2 only) | Pod is restarted (can result in lost work)
GPU | Deciding where to schedule the Pod //<br> Directly attaching GPU to container | Not applicable<br><small>(GPUs / [MIGs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html) / [vGPUs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/install-gpu-operator-vgpu.html) attach directly to a specific container)</small><br><small>([Time-Slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html) does not enforce limits)</small>

!!! warning "Resource Visibility"

    Typically, processes running in Pods __see ALL resources of their node__, regardless of what they request.
    Many tools will try and use all resources they can see (especially memory), you may want to [dedicate a node to each Notebook](#dedicated-notebook-nodes), rather than trying to share resources with other Pods on the same node.

#### CPU and Memory

When spawning a Notebook, users can request a certain amount of CPU and Memory, which directly set the `resources.requests` and `resources.limits` of the Pod's main container.

<div class="image-wrapper" markdown="1">
  ![Kubeflow Notebooks Spawner CPU/RAM (Dark Mode)](../../assets/images/kubeflow-notebooks-cpu-ram-DARK.png#only-dark)
  ![Kubeflow Notebooks Spawner CPU/RAM (Light Mode)](../../assets/images/kubeflow-notebooks-cpu-ram-LIGHT.png#only-light)
</div>

The following values configure CPU and Memory behavior for the Notebook Spawner UI:

Resource | Values
--- | ---
CPU | [`kubeflow_tools.notebooks.spawnerFormDefaults.cpu`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1525-L1535)
Memory | [`kubeflow_tools.notebooks.spawnerFormDefaults.memory`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1537-L1547)

For example, the following values set the default CPU request to `4.0` (with a limit factor of `1.2`) and the default Memory request to `32Gi` (with a limit factor of `1.2`):

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      cpu:
        ## uncomment to make the CPU field read-only
        #readOnly: true
        
        ## the default cpu request for the container
        value: "4.0"
        
        ## a factor by which to multiply the CPU request calculate the cpu limit
        ## (to disable cpu limits, set as "none")
        limitFactor: "1.2"

      memory:
        ## uncomment to make the RAM field read-only
        #readOnly: true
        
        ## the default memory request for the container
        value: "32Gi"
        
        ## a factor by which to multiply the memory request calculate the memory limit
        ## (to disable memory limits, set as "none")
        limitFactor: "1.2"
```

#### GPUs and Accelerators

When spawning a Notebook, users can request a certain number of GPUs (or other accelerators), which will set the `resources.limits` of the Pod's main container.
[__Kubernetes Device Plugins__](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/#using-device-plugins) are then able to attach the requested accelerators to the container.

<div class="image-wrapper" markdown="1">
  ![Kubeflow Notebooks Spawner GPUs (Dark Mode)](../../assets/images/kubeflow-notebooks-gpu-DARK.png#only-dark)
  ![Kubeflow Notebooks Spawner GPUs (Light Mode)](../../assets/images/kubeflow-notebooks-gpu-LIGHT.png#only-light)
</div>

The [`kubeflow_tools.notebooks.spawnerFormDefaults.gpus`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1549-L1573) values configure the GPU behavior for the Notebook Spawner UI.
For example, the following values set the default number of GPUs to `none`, and the default vendor to `nvidia.com/gpu`:

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      gpus:
        ## uncomment to make the GPU field read-only
        #readOnly: true
        
        value:
          ## the `limitKey` of the default vendor
          ## (to have no default, set as "")
          vendor: "nvidia.com/gpu"

          ## the list of available vendors in the dropdown
          vendors:
            - limitsKey: "nvidia.com/gpu"
              uiName: "NVIDIA"
          #  - limitsKey: "amd.com/gpu"
          #    uiName: "AMD"

          ## the default value of the limit
          ## (possible values: "none", "1", "2", "4", "8")
          num: "none"
```

!!! warning "Affinity and Tolerations"

    When using GPU nodes, you will likely also want to configure [node affinity](#affinity-and-anti-affinity) to select nodes based on labels (e.g. to choose a specific GPU model), and [node tolerations](#node-tolerations) (clusters usually have GPU nodes tainted to prevent non-GPU workloads from being scheduled on them).

!!! question_secondary "Can multiple Notebooks share a single GPU?"

    Nvidia provides a number of ways to share a single GPU between multiple Kubernetes Pods, these include 
    [MIGs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html), 
    [vGPUs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/install-gpu-operator-vgpu.html), 
    and [Time-Slicing](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html).
    Each method has different trade-offs, see [this blog post](https://developer.nvidia.com/blog/improving-gpu-utilization-in-kubernetes/) and the [Nvidia GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html) for more information.

    ...
    ...
    ... ?? can you select a mig right now, becaus we only allow requests of 1/2/4/8 GPUs ??
    ... ?? can it be done with an afinity ??
    ...

### __Storage Volumes__

Persistent file storage is provided using [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) which enable data to be kept between Notebook restarts and sometimes shared between multiple Notebooks.

When spawning a Notebook, users can choose to __create__ or __attach__ an existing _Persistent Volume Claim (PVC)_ as either a __Workspace Volume__ or a __Data Volume__.
Workspace Volumes are mounted at `/home/jovyan` and store the user's home directory.
Data Volumes can be mounted at any path, and are typically used to store datasets and other shared data.

The following values configure storage volume behavior for the Notebook Spawner UI:

Volume Type | Values
--- | ---
Workspace Volume | [`kubeflow_tools.notebooks.spawnerFormDefaults.workspaceVolume`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1575-L1597)
Data Volumes | [`kubeflow_tools.notebooks.spawnerFormDefaults.dataVolumes`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1599-L1620)

For example, the following values set the default Workspace Volume to use the `my-storage-class` StorageClass with a default size of `50Gi`:

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      workspaceVolume:
        value:
          ## WARNING: this should NOT be changed, ALL notebook images 
          ##          use `/home/jovyan` as their home directory
          mount: /home/jovyan

          ## NOTE: this section has the same structure as a PersistentVolumeClaim
          ##       https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#persistentvolumeclaim-v1-core
          newPvc:
            metadata:
              ## NOTE: "{notebook-name}" is replaced with the Notebook name
              name: "{notebook-name}-workspace"
            spec:
              ## NOTE: to use the cluster's default StorageClass, remove this line
              storageClassName: my-storage-class
              resources:
                requests:
                  storage: 50Gi
              accessModes:
                - ReadWriteOnce
```

!!! warning "StorageClass, AccessModes, and Performance"

    Notebooks are typically used for ML workloads, which are often IO-intensive.
    It is recommended to use a StorageClass that provides high-performance, typically this is only possible with drives which are attached to the node (rather than network-attached storage).
    
    Using node-attached drives will probably limit your [`accessModes`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes) to `ReadWriteOnce`, which means the volume can only be used by __one-or-more Pods__ running on a __single node__, unlike `ReadWriteMany` which can be used by __many Pods__ running on __many nodes__.

!!! info "No Default Workspace Volume"

    If you want users to always manually create/attach a Workspace Volume (rather than automatically creating a new one),
    you may set the `kubeflow_tools.notebooks.spawnerFormDefaults.workspaceVolume.value` to `null`:
    
    ```yaml
    kubeflow_tools:
      notebooks:
        spawnerFormDefaults:
          workspaceVolume:
            value: null
    ```

    ___WARNING:__ not having a default Workspace Volume increases the chance of users losing work by creating a Notebook without home directory persistence._

### __Affinity and Anti-Affinity__

Pod Affinity and Anti-Affinity are used to configure [how Pods are scheduled on nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity), and how they are [scheduled relative to other Pods](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity).

When spawning a Notebook, users can choose an "affinity config" from a predefined list (under the "Advanced Options" section at the bottom), which will set the `affinity` configs of the Pod's main container.
The [`kubeflow_tools.notebooks.spawnerFormDefaults.affinityConfig`](https://github.com/deployKF/deployKF/blob/v0.1.4/generator/default_values.yaml#L1622-L1657) values configure the available affinity configs for the Notebook Spawner UI.

For example, to allow selecting nodes based on labels (e.g. to choose a specific GPU model based on labels from [Nvidia GPU Feature Discovery](https://github.com/NVIDIA/gpu-feature-discovery)), you may use the following values:

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      affinityConfig:
        ## uncomment to make the affinity field read-only
        #readOnly: true
        
        ## The default value of the affinity
        value: ""

        ## The list of available affinities in the dropdown
        options:
            
          ## Nvidia A100 (40GB)
          - configKey: "nvidia_a100"
            displayName: "Nvidia A100"
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: "nvidia.com/gpu.product"
                          operator: "In"
                          values:
                            - "A100-SXM4-40GB"
  
          ## Nvidia V100 (32GB)
          - configKey: "nvidia_v100"
            displayName: "Nvidia V100"
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: "nvidia.com/gpu.product"
                          operator: "In"
                          values:
                            - "V100-SXM2-32GB"
  
          ## Nvidia GPU (with more than 16GB)
          - configKey: "nvidia_16gb"
            displayName: "Nvidia GPU (with more than 16GB)"
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: "nvidia.com/gpu.memory"
                          operator: "Gt"
                          values:
                            - "xxxxxxxxxxxxx"
```

### __Node Tolerations__

...

### __PodDefaults__

...

They can be selected under the `"Advanced Options"` -> `"Configurations"` section when spawning a Notebook.

---

## __Platform Configs__

Platform configs affect the entire platform, and are not exposed to users when spawning a Notebook.

### __Dedicated Notebook Nodes__

Because Notebook Pods are not well-behaved applications (from a resource perspective), a common pattern is to __provision a dedicated node__ for each Notebook Pod.
This can be achieved by using a combination of [Pod Affinity](#affinity-and-anti-affinity) and [Node Tolerations](#node-tolerations).

These steps show you how to configure dedicated nodes for Notebook Pods:

??? steps "Configure Dedicated Notebook Nodes"

    __Step 1: Prepare Node Pools__

    In the following example, we have 5 groups of nodes with different CPU/Memory configurations.

    We have configured the node pools as follows:

    - Each node pool is tainted with a different value of the `dedicated` key to __prevent other Pods from being scheduled on them__, and allow users to __choose which type of node they want__.
    - Each node is labeled with `lifecycle=kubeflow-notebook` to ensure Notebooks only consider our dedicated nodes (note, taints only repel, they do not attract).

    !!! warning "Cluster Autoscaler Required"

        Unless you want to have Nodes sitting idle most of the time, you will likely want to use a __node autoscaler__ to automatically provision and de-provision nodes based on the number of Notebook Pods:
    
    ??? question_secondary "How do I configure node autoscaling?"
    
        How you autoscale nodes will depend on your specific Kubernetes environment:
    
        - If you run Kubernetes on a managed cloud service, there is usually a built-in option for autoscaling node pools.
          Most cloud providers use the official [`cluster-autoscaler`](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler), but some specific options like [Karpenter](https://karpenter.sh/) (for AWS EKS) also exist.
        - If you run Kubernetes on-premises with [Rancher](https://ranchermanager.docs.rancher.com/), there are a number of VM-based [node drivers](https://ranchermanager.docs.rancher.com/v2.8/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/about-provisioning-drivers#node-drivers) including
          [vSphere](https://ranchermanager.docs.rancher.com/v2.8/how-to-guides/new-user-guides/launch-kubernetes-with-rancher/use-new-nodes-in-an-infra-provider/vsphere),
          [Nutanix](https://ranchermanager.docs.rancher.com/v2.8/how-to-guides/new-user-guides/launch-kubernetes-with-rancher/use-new-nodes-in-an-infra-provider/nutanix),
          and [Harvester](https://docs.harvesterhci.io/v1.2/rancher/node/node-driver)<sup>[[Open Source](https://github.com/harvester/harvester)]</sup>
          which can be [managed by `cluster-autoscaler`](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/rancher/README.md).    
    
    ---
    
    __Step 2: Configure Pod Affinity__
    
    We configure a single Pod Affinity config (for the Notebooks Spawner UI) to ensure only one Notebook Pod is scheduled on each Node, and to ensure that the Node has the correct label.
    
    - Using [`nodeAffinity`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) to require a Node with label `lifecycle=kubeflow-notebook`
    - Using [`podAntiAffinity`](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#types-of-inter-pod-affinity-and-anti-affinity) to require a Node WITHOUT an existing Pod having `notebook-name` label
    
    The following values configure the Affinity and Anti-Affinity in the Notebook Spawner UI:
    
    ```yaml
    kubeflow_tools:
      notebooks:
        spawnerFormDefaults:

          ## Affinity and Anti-Affinity
          ##
          affinityConfig:

            ## Set `readOnly` to `true` to ensure this affinity is always applied
            readOnly: true

            ## The default value of the affinity
            value: "dedicated_node_per_notebook"

            ## The list of available affinities in the dropdown
            options:
              - configKey: "dedicated_node_per_notebook"
                displayName: "Dedicated Node Per Notebook"
                affinity:

                  ## Node Affinity:
                  ##  - Require a Node with label `lifecycle=kubeflow-notebook`
                  nodeAffinity:
                    requiredDuringSchedulingIgnoredDuringExecution:
                      nodeSelectorTerms:
                        - matchExpressions:
                            - key: "lifecycle"
                              operator: "In"
                              values:
                                - "kubeflow-notebook"

                  ## Pod Anti-Affinity:
                  ##  - Require a Node WITHOUT an existing Pod having `notebook-name` label
                  podAntiAffinity:
                    requiredDuringSchedulingIgnoredDuringExecution:
                      - labelSelector:
                          matchExpressions:
                            - key: "notebook-name"
                              operator: "Exists"
                        topologyKey: "kubernetes.io/hostname"
                        ## NOTE: this makes the affinity work across namespaces
                        namespaceSelector: {}
    ```

    ---

    __Step 3: Configure Node Tolerations__

    We configure Node Tolerations to __allow Notebook Pods to be scheduled on the tainted nodes__, and __allow users to choose which type of node they want__.

    Here are the list of node taints assumed for this example:

    Key | Value | Effect
    --- | --- | ---
    `dedicated` | `kubeflow-c5.xlarge` | `NoSchedule`
    `dedicated` | `kubeflow-c5.2xlarge` | `NoSchedule`
    `dedicated` | `kubeflow-c5.4xlarge` | `NoSchedule`
    `dedicated` | `kubeflow-r5.8xlarge` | `NoSchedule`

    For example, the following values configure the Node Tolerations in the Notebook Spawner UI:

    ```yaml
    kubeflow_tools:
      notebooks:
        spawnerFormDefaults:

          ## Tolerations
          ##
          tolerationGroup:
            readOnly: false

            ## The default value of the toleration
            value: "group_1"

            ## The list of available tolerations in the dropdown
            options:
              - groupKey: "group_1"
                displayName: "4 CPU 8Gb Mem at ~$X.XXX USD per day"
                tolerations:
                  - key: "dedicated"
                    operator: "Equal"
                    value: "kubeflow-c5.xlarge"
                    effect: "NoSchedule"

              - groupKey: "group_2"
                displayName: "8 CPU 16Gb Mem at ~$X.XXX USD per day"
                tolerations:
                  - key: "dedicated"
                    operator: "Equal"
                    value: "kubeflow-c5.2xlarge"
                    effect: "NoSchedule"

              - groupKey: "group_3"
                displayName: "16 CPU 32Gb Mem at ~$X.XXX USD per day"
                tolerations:
                  - key: "dedicated"
                    operator: "Equal"
                    value: "kubeflow-c5.4xlarge"
                    effect: "NoSchedule"

              - groupKey: "group_4"
                displayName: "32 CPU 256Gb Mem at ~$X.XXX USD per day"
                tolerations:
                  - key: "dedicated"
                    operator: "Equal"
                    value: "kubeflow-r5.8xlarge"
                    effect: "NoSchedule"
    ```

    ---

    __Step 4: Configure CPU/Memory Resources__

    To prevent incompatible CPU/Memory resources, you may set a compatible value, and then set `readOnly` to `true` to prevent users from changing them.

    !!! info

        The value of the CPU and Memory __REQUESTS__ are not important, as the Notebook will be the only non-system Pod on the Node.
        However, it is important to set a value __less than your smallest Node type's capacity__, to ensure Kubernetes can schedule the Pod (while leaving enough resources for any system Pods).

        The value of the CPU and Memory __LIMITS__ should be __unset__, as we dont want Kubernetes to throttle or kill the Pod if it exceeds these limits.

    The following values configure the CPU and Memory resources in the Notebook Spawner UI:
    
    ```yaml
    kubeflow_tools:
      notebooks:
        spawnerFormDefaults:

          ## CPU Resources
          ##
          cpu:
            readOnly: true
            value: "1.0"
        
            ## NOTE: this is very important, as we don't want to limit the CPU
            limitFactor: "none"

          ## Memory Resources
          ##
          memory:
            readOnly: true
            value: "4Gi"

            ## NOTE: this is very important, as we don't want to limit the memory
            limitFactor: "none"
    ```

    !!! warning "GPUs"
    
        Because of how the [Nvidia GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html) works, the correct [GPU Resources](#gpus-and-accelerators) must still be set, even if the Pod is on a dedicated node.

    ---

    __Step 5: Test__

    Now, users should be able to choose a "node type" by choosing the corresponding "Toleration" group when spawning their Notebook.
    They will need to expand the `"Advanced Options"` section, and select the `"Toleration"` group which corresponds to the type of node they want.

    <div class="image-wrapper" markdown="1">
      ![Kubeflow Notebooks Dedicated Nodes (Dark Mode)](../../assets/images/kubeflow-notebooks-dedicated-nodes-DARK.png#only-dark)
      ![Kubeflow Notebooks Dedicated Nodes (Light Mode)](../../assets/images/kubeflow-notebooks-dedicated-nodes-LIGHT.png#only-light)
    </div>

    Depending on your cluster's autoscaler settings, you should see a new node being provisioned when a Notebook is spawned, and the node being de-provisioned when the Notebook is deleted.

### __Idle Notebook Culling__

Kubeflow Notebooks supports automatically culling idle Notebook Pods, which is configured by the [`kubeflow_tools.notebooks.notebookCulling`](https://github.com/deployKF/deployKF/blob/v0.1.1/generator/default_values.yaml#L1577-L1588) values.

!!! warning "Jupyter Notebooks Only"

    Currently, only Jupyter Notebooks are supported for idle culling, see the upstream [design proposal](https://github.com/kubeflow/kubeflow/blob/master/components/proposals/20220121-jupyter-notebook-idleness.md) for more information.

The following values configure the idle culling settings:

- .....

For example, the following values will enable idle culling after 1 day of inactivity:

```yaml
kubeflow_tools:
  notebooks:
    notebookCulling:
      enabled: true
      idleTime: 1440 # 1 day in minutes
```


### __Kubeflow Pipelines Authentication__

...

The [`kubeflow_tools.pipelines.profileResourceGeneration.kfpApiTokenPodDefault`](https://github.com/deployKF/deployKF/blob/v0.1.1/generator/default_values.yaml#L1803-L1811) value 
configures if a `PodDefault` named `"kubeflow-pipelines-api-token"` is automatically generated in each profile namespace.

If the user selects this "configuration" when spawning their Notebook, they will be able to use the __Kubeflow Pipelines Python SDK__ from the Notebook without needing to manually authenticate.

To have this "configuration" selected by default in the spawner, you may use the following values:

```yaml
kubeflow_tools:
  notebooks:
    spawnerFormDefaults:
      configurations:
        value:
          - "kubeflow-pipelines-api-token"
```

For more information, see the [Access Kubeflow Pipelines API](../../user-guides/access-kubeflow-pipelines-api.md#kubernetes-serviceaccount-token) user guide.


### __Override Notebook Template__

!!! warning

    This is an advanced feature, and should be used with caution.

...