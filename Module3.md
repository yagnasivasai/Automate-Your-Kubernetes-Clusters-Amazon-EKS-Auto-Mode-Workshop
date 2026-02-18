Module 3 - Cluster Upgrades
Understanding Kubernetes upgrades in EKS
Amazon Elastic Kubernetes Service (EKS) requires careful planning for upgrades. The upstream Kubernetes project (Kubernetes Releases ) undergoes continuous improvement, with regular updates introducing new functionalities, design enhancements, and bug corrections. Minor version releases typically occur every four months, and each version receives community support for approximately 1 year  following its launch.

Why keep EKS updated?
Maintaining current Amazon EKS versions is crucial for:

Security: Protecting Kubernetes clusters against vulnerabilities
Stability: Ensuring reliable performance and compatibility
Innovation: Accessing the latest platform features and capabilities
For detailed information about EKS versions and updates, refer to the Amazon EKS Kubernetes versions documentation .

EKS Auto Mode: enhanced shared responsibility
A Kubernetes deployment consists of both control plane and data plane components (worker nodes).

Amazon EKS has always managed the Kubernetes control plane and upgrades. Before Amazon EKS Auto Mode, cluster owners were responsible for initiating upgrades for both the control plane and data plane. This includes upgrading worker nodes in Self Managed node groups, Managed Node Groups, and other add-ons.

Amazon EKS Auto Mode represents a significant advancement in Kubernetes cluster management, expanding AWS's responsibility to include the automatic upgrading of cluster infrastructure. This enhancement covers both worker nodes and core cluster capabilities, substantially reducing the operational burden on customers.

To illustrate the shared responsibility model under Amazon EKS Auto Mode, please refer to the image below: Shared responsibility model with auto mode

Component Management
Red areas (the larger portion of the diagram): represent components fully managed by AWS
Green areas (the smaller portion of the diagram): indicate the aspects that remain under customer management
Kubernetes Version Structure
From the Kubernetes versioning documentation : Versions are expressed as x.y.z, where:

x: Major version
y: Minor version (Released every ~4 months)
z: Patch version (Monthly releases)
The Kubernetes version indicates both control-plane (apiserver, controller-manager, etc.) and data-plane (e.g., kubelet, kube-proxy, etc.) components.

Amazon EKS Version Management
Amazon EKS provides a managed Kubernetes control plane with a range of supported versions :

Standard Support: 14 months
Extended Support: Additional 12 months (optional with additional costs, which can be disabled )
Version Compatibility: Data-plane components (e.g., kubelet, kube-proxy) should match the control plane minor version but can also stay on lower versions up to the allowed version skew policy . When using node auto-scalers such as Karpenter, the AMI minor version can be automatically discovered to match the EKS Control plane version.
Amazon EKS platform versions - Kubernetes control-plane patch versions upgrade
Amazon EKS periodically releases new platform versions  to enable new control plane settings and provide security fixes. Each Kubernetes minor version may have multiple associated platform versions. These EKS Platform versions are automatically upgraded in a gradual rollout process with no manual intervention required from the customer. Importantly, new Amazon EKS platform versions don't introduce breaking changes or cause service interruptions.

Staying current with the latest Kubernetes minor version is crucial for maintaining a secure and efficient EKS environment. This approach aligns with the shared responsibility model in Amazon EKS, ensuring that clusters run with the latest security patches and bug fixes, thereby reducing security vulnerabilities. Additionally, it offers improved performance, scalability, and reliability, ultimately providing better service to applications and customers.

Example
If a cluster runs Kubernetes 1.25, AWS might update the platform version from eks.1 to eks.2 automatically to apply security patches while maintaining the same Kubernetes version.

Add-ons Management
The last part of an upgrade process is to upgrade the add-ons  which are system wide applications that provide additional capabilities for other business applications that run on the cluster. For example:

Observability tools
Security implementations
Networking solutions
Storage integrations
Amazon EKS provides additional guidance on upgrades as part of the EKS best practices guide which you can find in the documentation .

After understanding the Kubernetes version cadence and compatibility, next we'll experiment with how Amazon EKS Auto Mode manages zero touch upgrades of cluster infrastructure that includes nodes and the core capabilities.

Understanding upgrades with Amazon EKS Auto Mode
Overview | Upgrade flow process | Time controls | Summary

Overview
Cluster upgrade flow Amazon EKS Auto Mode revolutionizes Kubernetes cluster management by providing zero-touch updates for our entire cluster infrastructure.

Upgrade flow process
1. Version Verification
Check the latest available Amazon EKS version and verify compatibility with current applications
Initiate a version update when needed
2. Control Plane and cluster component updates
Initiate control plane version upgrade
With Auto Mode, any of the managed cluster capabilities will also be automatically updated to ensure compatibility across versions
3. Data Plane Update
Node rolling update Graceful node updates follow this process:

Node replacement with latest AMIs using Karpenter's Drift management capability
Rolling update strategy
Respect for Pod Disruption Budgets (PDBs)
Time controls
Expiration Settings
Default: worker nodes will be fully automatically upgraded no later than 14 days
Custom NodePools: Up to 21 days
Termination Grace Period
Amazon EKS Auto Mode sets a default of 24 hours for terminationGracePeriod. This period defines the amount of time a Node can be drained before Karpenter forcibly cleans up the node. During draining, pods blocking eviction (such as those with PDBs and do-not-disrupt annotations) will be respected until the terminationGracePeriod is reached, after which those pods will be forcibly deleted.

Disruption Controls
We can limit the rate at which Amazon EKS Auto Mode disrupts nodes through the NodePool's spec.disruption.budgets. Disruption budgets allow us to control the rate of worker node upgrades or ensure upgrades only happen during specific dates and times (using schedule).

Disruption budgets can be configured for the following disruption reasons:

Empty - when a node is has no running pods
Underutilized - when a node can be removed or replaced with a smaller instance type
Drifted - when a node has diverged from its desired state (for example when the EKS control plane version differs from worker node version)
By default, the Amazon EKS Auto Mode general-purpose NodePool has the following disruption budget:

...
spec:
  disruption:
    consolidateAfter: 0s
    consolidationPolicy: WhenEmptyOrUnderutilized
    budgets:
    - nodes: 10%
...
With this budget, Amazon EKS Auto Mode will disrupt at most 10% of active nodes at a time.

...
spec:
  disruption:
    consolidateAfter: 0s
    consolidationPolicy: WhenEmptyOrUnderutilized
    budgets:
    - nodes: 10%
    - schedule: 0 9 * * mon-fri
      duration: 8h
      nodes: "0"
      reasons:
      - Drifted
...
With this configuration, we limit Amazon EKS Auto Mode from disrupting nodes ("0") during weekday business hours if Drift is triggered, while allowing at most 10% of active nodes to be disrupted at all other times.

Summary
Now that we understand how Amazon EKS Auto Mode handles upgrades, let's proceed to upgrading the cluster to the most recent Kubernetes version.

Upgrading the cluster
Upgrade | Summary

Upgrade the cluster
In this section of the workshop, we'll gain hands-on experience with Amazon EKS Auto Mode's automatic cluster infrastructure updates. We'll begin by updating the control plane.

Check Current Versions
➤ First, let's check the current Amazon EKS control plane version:

aws eks describe-cluster --region $AWS_REGION --name $DEMO_CLUSTER_NAME --query "cluster.version" --output text

This is the output:

1.32
We can see that our Amazon EKS control plane is running Amazon EKS version 1.32.

➤ Now, let's observe the current worker nodes (data plane) version:

kubectl get nodes

The output should be similar to the following:

NAME                  STATUS   ROLES    AGE   VERSION
i-01c27cd2a458478a7   Ready    <none>   78m   v1.32.3-eks-156189c
i-06a7bbee7d6f29f45   Ready    <none>   56m   v1.32.3-eks-156189c
...
We can see that the worker nodes are also running Amazon EKS version 1.32.

Look for deprecated APIs and new versions for additional components
Under the shared responsibility model, we should review the Kubernetes release  page, and for each version we're upgrading to, verify that there are no API deprecations or changes that might affect our business applications.

Amazon EKS also provides the Upgrade Insights  tool (both in the AWS EKS console under the cluster's Observability dashboard and via the AWS API). This tool allows you to identify issues that could impact your ability to upgrade to new versions of Kubernetes.

Additionally, we need to check for new versions of any 3rd party or community add-ons installed in our cluster, and upgrade them after completing the EKS cluster upgrade.

Upgrade the control plane
Let's upgrade our Amazon EKS control plane version to 1.33.

Note: While there are multiple ways to upgrade an Amazon EKS control plane (using eksctl, AWS Console, AWS CLI, or Infrastructure as Code), we'll use the AWS CLI for simplicity in this workshop.

➤ Execute the following command:

aws eks update-cluster-version --region $AWS_REGION --name $DEMO_CLUSTER_NAME --kubernetes-version 1.33

You will see output similar to this:

{
    "update": {
        "id": "7e15217f-3ed4-3eaa-91e2-b1fc05bd1c89",
        "status": "InProgress",
        "type": "VersionUpdate",
        "params": [
            {
                "type": "Version",
                "value": "1.33"
            },
            {
                "type": "PlatformVersion",
                "value": "eks.4"
            }
        ],
        "createdAt": "2025-06-08T16:32:24.518000+00:00",
        "errors": []
    }
}
In the output, we can see the Amazon EKS version is being updated to 1.33. Also note the PlatformVersion parameter which is documented in the Amazon EKS platform version  documentation. At the time of creating this workshop, this output is eks.4, but it may differ in future platform versions of Amazon EKS.

Summary
In this section we've initiated an upgrade of the Amazon EKS Auto Mode cluster control plane from version 1.32 to 1.33. With Auto Mode, AWS will handle the worker node upgrades automatically, demonstrating the zero-touch infrastructure update capability that makes Kubernetes operations simpler.

Checking the upgrade
Check the upgrade | Summary

Check the upgrade
➤ Using the aws eks describe-cluster AWS CLI command, we will check if the upgrade process is complete:

aws eks describe-cluster --region $AWS_REGION --name $DEMO_CLUSTER_NAME --query "cluster.version" --output text

If enough time has passed, we should see that our cluster is now have been upgraded and is now running Amazon EKS version 1.33:

1.33
If by this point the control plane haven't yet been upgraded, and depending on your available time, we advise you to either take a small break here, or progress to the Migration patterns module, and get back to check the upgrade once you're done.

For convenience, you can open a new IDE terminal and execute the following command:

watch -t aws eks describe-cluster --region $AWS_REGION --name $DEMO_CLUSTER_NAME --query "cluster.version" --output text

➤ We can now verify the worker nodes version:

kubectl get nodes

If the worker nodes have been upgraded we should see their version matching the Amazon EKS version:

NAME                  STATUS   ROLES    AGE     VERSION
i-01774cb623e95980b   Ready    <none>   6m33s   v1.33.0-eks-987fa8d
...
Similarly, if by this point the nodes haven't yet been upgraded, you can open a new IDE terminal and execute the following command:

watch -t kubectl get nodes

Once Amazon EKS Auto Mode detects that the EKS control plane version was upgraded, and it triggered node replacement through its drift detection mechanism. Let's examine the related events that occurred during this process.

➤ Execute the following command to view the drift-related events:

kubectl get events | grep Drifted

19m         Normal    DisruptionLaunching              nodeclaim/advanced-networking-vpthn                               Launching NodeClaim: Drifted
32s         Normal    DisruptionTerminating            nodeclaim/advanced-networking-vpthn                               Disrupting NodeClaim: Drifted
19m         Normal    DisruptionTerminating            node/i-03e6de8867bb6e67e                                          Disrupting Node: Drifted
2m2s        Normal    DisruptionTerminating            node/i-04a21a34422cd8e7d                                          Disrupting Node: Drifted
...
This output shows the orchestrated node replacement process in action. Amazon EKS Auto Mode follows a carefully sequenced approach:

First, it launches new replacement nodes
Then it evicts pods from the old nodes
It waits until the new replacement nodes are in the Ready state before terminating the old nodes
When Pod Disruption Budgets (PDBs) are configured, Auto Mode respects these budgets by using a backoff retry eviction strategy. Pods will never be forcibly deleted during normal operation. However, pods that fail to shut down will prevent a node from being deleted until the terminationGracePeriod is reached, at which point those pods will be forcibly deleted.

Summary
Congratulations! In this section, we have successfully updated our cluster to the latest version of Kubernetes. We've observed how Amazon EKS Auto Mode automatically manages the worker node upgrades to match the control plane version, demonstrating the zero-touch infrastructure update capabilities that simplify Kubernetes operations.
