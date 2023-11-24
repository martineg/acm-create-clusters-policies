# Create and managed Clusters using ACM and GitOps  

This repository contains example configuration enabling creating clusters with explicit customizations and an opinionated reusable base.

### File and directory descriptions

There are 2 main directories: `base` and `clusters`
- The "**base**" directory contains common configuration needed to create a cluster regardless of which cloud you are creating the cluster on.  And Cloud specific configuration.
  - The Cloud specific configuration includes the common configuration.  Using kustomize to patch the common objects with Cloud specific details.
- The "**clusters**" directory contains configuration for each cluster to be created and managed.  
  - The cluster configuration includes the Cloud specific configuration.
  - Each cluster definition is transformed by kustomize before it is added to the policy in the PolicyGenerator

There is an example Credentials secret in the "**hub-data**" directory to showcase the required objects that need to be on the hub.  This repository expects the Credentials and Placement details to already be on the Hub or to be otherwise managed.  You could add the required Placements to this repository to make a more complete process.

### Adding a new cluster
To add a new cluster requires a few simple steps.
1. Create a directory for the cluster under the  "**clusters**" directory.  Naming the directory after the name of the cluster makes management easier later.
2. This directory should contain the `install-config-secret`, `managedcluster`, and `kustomization` file to complete the assembly of the objects that define the cluster.
3. Add the directory path to the PolicyGenerator

#### Cluster file explanations
A more indepth review of the files needed in the cluster directory
- **managedcluster.yml**
  This will be merged with the standard ManagedCluster.  Here we will specify labels that should be applied to this specific cluster.  Doing so allows us to managed the labels on the cluster with a GitOps process.
  ```
  apiVersion: cluster.open-cluster-management.io/v1
  kind: ManagedCluster
  metadata:
    name: cluster-name  ## needs to be present, but does not matter
    labels:
      storage: odf
  ```

- **install-config-secret.yml**
  This will use Policy templating to fill-in details from the ACM Credentials that is already on the hub.
  It is important the credentials lookup is correct, along with the cluster name.  This name needs to match what is specified in the kustomization files.
  Here you can specify the complete install-config as needed.
  ```
  apiVersion: policy.open-cluster-management.io/v1
  kind: ConfigurationPolicy
  metadata:
    name: install-config-secret
  spec:
    remediationAction: enforce
    severity: low
    object-templates-raw: |
      {{- $vc := (lookup "v1" "Secret" "cluster-provider-infra" "vsphere-creds") }}
      {{- $installConfig := `
      apiVersion: v1
      baseDomain: '%[1]s'
      compute:
      - architecture: amd64
        hyperthreading: Enabled
        name: worker
        platform:
          vsphere:
            coresPerSocket: 2
            cpus: 4
            memoryMB: 16384
            osDisk:
              diskSizeGB: 120
        replicas: 2
      controlPlane:
        architecture: amd64
        hyperthreading: Enabled
        name: master
        platform:
          vsphere:
            coresPerSocket: 2
            cpus: 4
            memoryMB: 32768
            osDisk:
              diskSizeGB: 120
        replicas: 3
      metadata:
        name: ocp01
      networking:
  ```

- **kustomization.yaml**
  Contains information to assemble all of the cluster objects and provide the appropriate transformations.
  The namespace must match the name of the cluster.
  The `resources` block must include the cloud base you are deploying the cluster to
  The generated ConfigMap contains attributes needed to build the cluster.
  If you wish to include additional customizations to the output objects you can add them to the patches block similar to how the ManagedCluster is.  This is demonstrated in the bry-aws cluster with adding the zones to the MachinePool.
  ```
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization

  ## specify the namespace to match the name of the cluster.
  ## this set the namespace on objects included in the policy, not the policy itself
  namespace: ocp01

  ## include any additional objects that should be part of the policy
  resources:
    - ../../base/vsphere/
    - install-config-secret.yml

  ## Set cluster values for clusterset and imageset.
  ## If you want a custom ImageSet - add a custome file to resources above and set the name below
  ## The transformer below includes these values in the final policy
  generatorOptions:
  disableNameSuffixHash: true

  configMapGenerator:
    - name: set-cluster-values
      literals:
      - clusterSet=prod
      - imageSetRefName=img4.13.16-multi-appsub
      - autoScaleMaxWorkerNodes=5


  ## patch the managed cluster to define cluster specific lables
  patches:
    - target:
        kind: ManagedCluster
      path: managedcluster.yml

  transformers:
    - ../../base/common/kustomize-configs/
  ```

