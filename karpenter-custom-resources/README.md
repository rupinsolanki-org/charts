# Karpenter Custom Resources Helm Chart

This Helm chart facilitates the deployment of Karpenter `EC2NodeClass` and `NodePool` custom resources on an AWS EKS cluster where Karpenter is already installed. It allows for declarative management of these Karpenter configurations through Helm values.

**Important:** This chart *only* deploys `EC2NodeClass` and `NodePool` custom resources. It *does not* install Karpenter itself. You must have Karpenter installed and configured in your EKS cluster before using this chart.

## Prerequisites

*   Helm 3+
*   Kubernetes EKS cluster
*   Karpenter installed and configured on the EKS cluster. (See [Karpenter Documentation](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/))
*   Appropriate IAM roles and instance profiles for Karpenter nodes (e.g., `KarpenterNodeRole-YourClusterName`).
*   Subnets and security groups tagged for Karpenter discovery (e.g., `karpenter.sh/discovery: YourClusterName`).

## Chart Structure

*   **`values.yaml`**: Defines the `EC2NodeClass` and `NodePool` configurations.
*   **`schemas/`**: Contains JSON schemas for validating the structure of `values.yaml`.
    *   `ec2nodeclass-schema.json`: Schema for `EC2NodeClass` objects.
    *   `nodepool-schema.json`: Schema for `NodePool` objects.
    *   `values.schema.json`: Main schema referencing the above for validating the entire `values.yaml`.
*   **`templates/`**: Contains the Helm templates.
    *   `ec2nodeclass.yaml`: Template for generating `EC2NodeClass` Kubernetes manifests.
    *   `nodepool.yaml`: Template for generating `NodePool` Kubernetes manifests.
    *   `_helpers.tpl`: Common Helm helper templates.
*   **`Chart.yaml`**: Metadata for the Helm chart.

## Installation

1.  **Add Repository (if applicable)**
    If this chart is packaged and hosted in a Helm repository, add the repository first:
    ```bash
    helm repo add <your-repo-name> <your-repo-url>
    helm repo update
    ```

2.  **Prepare `values.yaml`**
    Create a `custom-values.yaml` (or any name you prefer) file and define your `EC2NodeClass` and `NodePool` resources. Refer to the default `values.yaml` in this chart for the structure and examples.

    **Example `custom-values.yaml`:**
    ```yaml
    global:
      clusterName: "my-eks-cluster" # Replace with your actual EKS cluster name

    ec2NodeClasses:
      - name: "default-linux"
        amiFamily: "AL2"
        role: "KarpenterNodeRole-my-eks-cluster" # Ensure this role exists
        subnetSelectorTerms:
          - tags:
              karpenter.sh/discovery: "my-eks-cluster"
        securityGroupSelectorTerms:
          - tags:
              karpenter.sh/discovery: "my-eks-cluster"
        tags:
          intent: "general-linux-workloads"
          provisioner: "karpenter"
        kubelet:
          maxPods: 110
          # clusterDNS: ["10.100.0.10"] # Replace with your cluster's DNS IP

    nodePools:
      - name: "general-purpose"
        ec2NodeClassName: "default-linux" # Must match an ec2NodeClasses.name
        template:
          metadata:
            labels:
              team: "backend"
              environment: "dev"
          spec:
            requirements:
              - key: "karpenter.sh/capacity-type"
                operator: In
                values: ["spot", "on-demand"]
              - key: "karpenter.k8s.aws/instance-category"
                operator: In
                values: ["m", "c", "r", "t"]
              - key: "kubernetes.io/arch"
                operator: In
                values: ["amd64"]
              - key: "kubernetes.io/os"
                operator: In
                values: ["linux"]
            expireAfter: "168h" # Nodes expire after 7 days
        disruption:
          consolidationPolicy: WhenEmptyOrUnderutilized
          consolidateAfter: "30s"
        limits:
          cpu: "500"
          memory: "2000Gi"
    ```

3.  **Install the Chart**
    ```bash
    helm install <release-name> <chart-path-or-repo/chart-name> -f custom-values.yaml -n karpenter
    ```
    Replace `<release-name>` with your desired release name (e.g., `karpenter-resources`), `<chart-path-or-repo/chart-name>` with the path to this chart directory or `your-repo-name/karpenter-custom-resources` if using a repo, and `-n karpenter` to install in the `karpenter` namespace (recommended).

    Example (installing from local path):
    ```bash
    helm install karpenter-config ./karpenter-custom-resources -f custom-values.yaml -n karpenter
    ```

## Configuration

The primary configuration is done via the `values.yaml` file.

### Top-Level Values

*   `global.clusterName` (string, optional): Your EKS cluster name. Used in some default selectors.
*   `ec2NodeClasses` (list of objects): Defines your EC2NodeClass resources.
*   `nodePools` (list of objects): Defines your NodePool resources.

### `ec2NodeClasses` Configuration

Each item in the `ec2NodeClasses` list is an object representing an `EC2NodeClass`. Key fields include:

*   `name` (string, required): Name of the EC2NodeClass. This will be the `metadata.name`.
*   `amiFamily` (string, optional): E.g., "AL2", "Bottlerocket", "Custom". Required if not using `amiSelectorTerms.alias`.
*   `amiSelectorTerms` (list, optional): For selecting AMIs (tags, id, name, owner, alias, ssmParameter).
*   `subnetSelectorTerms` (list, required): For selecting subnets (tags, id).
*   `securityGroupSelectorTerms` (list, required): For selecting security groups (tags, id, name).
*   `role` (string, conditional): IAM role for nodes. Required if `instanceProfile` is not set.
*   `instanceProfile` (string, conditional): IAM instance profile. Required if `role` is not set.
*   `tags` (object, optional): Additional tags for EC2 resources.
*   `kubelet` (object, optional): Kubelet configuration (e.g., `maxPods`, `podsPerCore`, `systemReserved`, `kubeReserved`, `evictionHard`).
*   `metadataOptions` (object, optional): Instance metadata service (IMDS) options.
*   `blockDeviceMappings` (list, optional): EBS volume configurations.
*   `detailedMonitoring` (boolean, optional): Enable detailed EC2 monitoring.
*   `associatePublicIPAddress` (boolean, optional): Whether to associate public IP.
*   `userData` (string, optional): Custom EC2 user data.
*   `instanceStorePolicy` (string, optional): E.g., "RAID0".
*   `capacityReservationSelectorTerms` (list, optional): For using EC2 Capacity Reservations.

Refer to the [official Karpenter EC2NodeClass documentation](https://karpenter.sh/docs/concepts/nodeclasses/) and the provided `values.yaml` and `schemas/ec2nodeclass-schema.json` for all available fields and their structures.

### `nodePools` Configuration

Each item in the `nodePools` list is an object representing a `NodePool`. Key fields include:

*   `name` (string, required): Name of the NodePool. This will be the `metadata.name`.
*   `ec2NodeClassName` (string, required): The name of an `EC2NodeClass` defined in `ec2NodeClasses` to link to.
*   `weight` (integer, optional): Priority for scheduling. Higher is more preferred.
*   `template.metadata.labels` (object, optional): Labels to apply to nodes provisioned by this pool.
*   `template.metadata.annotations` (object, optional): Annotations for nodes.
*   `template.spec.taints` (list, optional): Taints to apply to nodes.
*   `template.spec.startupTaints` (list, optional): Taints applied at startup, expected to be removed.
*   `template.spec.expireAfter` (string, optional): Node lifetime (e.g., "720h" or "Never").
*   `template.spec.terminationGracePeriod` (string, optional): Duration for node draining before forced deletion.
*   `template.spec.requirements` (list, required): Constraints for provisioned nodes (instance type, zone, arch, OS, capacity type, etc.).
*   `disruption` (object, optional): Disruption settings.
    *   `consolidationPolicy` (string): "WhenEmptyOrUnderutilized" or "WhenEmpty".
    *   `consolidateAfter` (string): E.g., "30s" or "Never".
    *   `budgets` (list): Disruption budgets to control scale-down velocity.
*   `limits` (object, optional): Resource limits for the pool (e.g., `cpu: "1000"`, `memory: "1000Gi"`, `"nvidia.com/gpu": "4"`).

Refer to the [official Karpenter NodePool documentation](https://karpenter.sh/docs/concepts/nodepools/) and the provided `values.yaml` and `schemas/nodepool-schema.json` for all available fields.

### Linking NodePools to EC2NodeClasses

A `NodePool` is linked to an `EC2NodeClass` via the `ec2NodeClassName` field in the NodePool definition. This name must exactly match the `name` field of one of the `EC2NodeClass` definitions in the `ec2NodeClasses` list.

The Helm chart translates this into the required `spec.template.spec.nodeClassRef` in the generated NodePool manifest:
```yaml
# NodePool spec excerpt
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws # Hardcoded in template
        kind: EC2NodeClass       # Hardcoded in template
        name: <value from nodePools.ec2NodeClassName>
```

## Validation

This chart includes JSON schemas to help validate your `values.yaml` file. You can use tools like `helm lint` or IDE plugins that support JSON schema validation.

To lint the chart with schema validation (Helm v3.7+):
```bash
helm lint ./karpenter-custom-resources -f custom-values.yaml
```
If your Helm version is older or for more specific validation, you might use a tool like `kubeconform` with the schemas, or `yamale`.

## Upgrading

To upgrade a release:
```bash
helm upgrade <release-name> <chart-path-or-repo/chart-name> -f custom-values.yaml -n karpenter
```

## Uninstallation

To uninstall/delete the `<release-name>` deployment:
```bash
helm uninstall <release-name> -n karpenter
```
This will delete the `EC2NodeClass` and `NodePool` resources created by this chart. It will not affect running nodes managed by Karpenter unless those nodes were solely defined by the resources being deleted and Karpenter subsequently deprovisions them.

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

Specify your license here (e.g., Apache 2.0).
