# Test Plan for Upgrading Red Hat OpenShift AI (RHOAI)

## Motivation

In the MOC, we do frequent upgrades of RHOAI. Every time we upgrade, we do testing in our test cluster of the new version. However, we don't have an explicit test plan for this. We just test whether things like model serving, pipelines, spinning up workbenches, etc is working. We need to have an actual reproducible upgrade plan. 

### Operator Health

Make sure all Operators are installed and in ready state. Validate that the operator Deployment is `Available`.

| Operator | Deployment Name | Namespace |
|----------|-----------------|-----------|
| Red Hat OpenShift AI | `rhods-operator` | `redhat-ods-operator` |
| Node Feature Discovery (NFD) | `nfd-controller-manager` | `openshift-nfd` |
| NVIDIA GPU Operator | `gpu-operator` | `nvidia-gpu-operator` |
| OpenShift Serverless | `knative-openshift` | `openshift-serverless` |
| OpenShift Service Mesh | `istio-operator` | `openshift-operators` |

### Configuration & Component Readiness

All configurations need to be in a Ready state. I define a configuration as anything that needs to be applied after an operator is installed to get system to working state. 

#### Configurations

| Configuration | API/Kind | Expected State |
|---------------|----------|----------------|
| DataScienceCluster | `datasciencecluster.opendatahub.io/v1` | `phase: Ready` |
| GPU ClusterPolicy | `nvidia.com/v1/ClusterPolicy` | `state: ready` |
| NFD Instance | `nfd.openshift.io/v1/NodeFeatureDiscovery` | `Available: True` |

#### DataScienceCluster Components

All components should have their respective `Ready` condition set to `True`.

| Component | Condition Type |
|-----------|----------------|
| Dashboard | `DashboardReady` |
| Workbenches | `WorkbenchesReady` |
| DataSciencePipelines | `DataSciencePipelinesReady` |
| CodeFlare | `CodeFlareReady` |
| KServe | `KserveReady` |
| Ray | `RayReady` |
| TrustyAI | `TrustyAIReady` |
| ModelMeshServing | `ModelMeshServingReady` |
| Kueue | `KueueReady` |

### GPU Validation

#### Node Labels
- Ensure all GPU nodes have NFD labels applied
    - `feature.node.kubernetes.io/pci-10de.present` or `feature.node.kubernetes.io/pci-0302_10de.present`
- Ensure all GPU nodes have NVIDIA GPU Operator labels
    - `nvidia.com/gpu.present`
    - `nvidia.com/gpu.count`

#### Required Pods Running
| Pod Type | Namespace | Label Selector |
|----------|-----------|----------------|
| NFD Worker | `openshift-nfd` | `app=nfd-worker` |
| NVIDIA GPU Driver | `nvidia-gpu-operator` | `app.kubernetes.io/component=nvidia-driver` |

### Model Serving

- Deploy a model using the vLLM ServingRuntime for KServe on a GPU
    - Verifies proper GPU node scheduling
    - Verifies KServe/Serverless/Service Mesh integration
- Test inference endpoint accessibility
    - Without Token Authentication
    - With Token Authentication enabled
- Test route types
    - External route (accessible outside cluster)
    - Internal route (cluster-internal only)

### Pipelines

- Import a pipeline from RHOAI UI
- Import a pipeline from within a workbench 
- Pipeline run working as expected
    - Run starts successfully
    - Run completes without errors
    - Can view logs for each step
    - Artifacts are stored correctly (S3)

### Workbenches

- Spin up workbench using RHOAI UI
    - Standard workbench (CPU only)
    - GPU-enabled workbench
        - Verify GPU is visible inside notebook (`nvidia-smi`)


### Connections and Storage

- Create S3 connection within a Data Science Project
- Attach S3 connection to a workbench
- Use S3 connection for model deployment (model stored in S3)
- Verify PVC provisioning for workbenches

### Authentication & Authorization
- RHOAI Dashboard accessible via OpenShift OAuth
- User permissions working correctly (admin vs regular user)

### Networking
- Service Mesh (Istio) control plane healthy


### Monitoring & Observability
- GPU metrics visible (DCGM exporter)

