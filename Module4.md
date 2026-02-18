Module 4 - Migrate an Amazon EKS Cluster to EKS Auto Mode
In this module, we will explore migration strategies from an existing Amazon EKS cluster to EKS Auto Mode.

We will use an existing Amazon EKS cluster with several different compute options for the worker nodes that host the cluster's applications and operational software, and gradually migrate from each of these options to Amazon EKS Auto Mode.

Cluster configuration
Setup | Overview | Cluster Add-ons | Self-Managed Karpenter Configuration | Demo Application

Setup
During the provisioning of this workshop we've created several Amazon EKS clusters, with one specifically intended for this module.

➤ Before we begin, let's switch the current kubeconfig context by executing:

kubectl config use-context arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${MIGRATION_CLUSTER_NAME}

You should receive an output similar to:

Switched to context "<a cluster ARN>".
Verify the cluster setup by executing the following command:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The command above shows the distribution of compute instances in the cluster across compute options and should look similar to the following:

fargate-ip-192-168-116-56.us-west-2.compute.internal   |  us-west-2c  |  FARGATE    |  profile:    apps
fargate-ip-192-168-180-168.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
fargate-ip-192-168-191-115.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
ip-192-168-100-194.us-west-2.compute.internal          |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-155-246.us-west-2.compute.internal          |  us-west-2a  |  KARPENTER  |  nodepool:   apps
...
ip-192-168-173-151.us-west-2.compute.internal          |  us-west-2b  |  KARPENTER  |  nodepool:   apps
ip-192-168-28-67.us-west-2.compute.internal            |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-51-140.us-west-2.compute.internal           |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-83-58.us-west-2.compute.internal            |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-26-158.us-west-2.compute.internal           |  us-west-2c  |  ON_DEMAND  |  system-mng  
ip-192-168-33-46.us-west-2.compute.internal            |  us-west-2a  |  ON_DEMAND  |  system-mng  
ip-192-168-86-255.us-west-2.compute.internal           |  us-west-2b  |  ON_DEMAND  |  system-mng  
Note that self-managed Karpenter nodes distribution may differ due to the dynamic nature of Karpenter provision and consolidation (hence ... in the output above), but the nodes should follow the same approximate distribution.

Overview
The compute options in the cluster, as shown in the output above, include:

An AWS Fargate profile  that allows us to target specific applications to be scheduled on AWS Fargate . These are the lines marked by profile: apps.
Several Amazon EKS managed node groups  that host the cluster operational software and some of the applications. These are the lines marked by system-mng for the cluster-critical managed node group and apps-mng for the applications' managed node group.
Self-managed Karpenter  nodes that host the rest of the applications in the cluster. These are marked by nodepool: apps.
The cluster also contains the required Amazon EKS add-ons :

The Amazon VPC CNI plugin for Kubernetes  add-on, which provides native VPC networking for the cluster
The CoreDNS  add-on, which serves as the Kubernetes cluster DNS server
The Kube-proxy  add-on, which maintains network rules on each Amazon EC2 worker node
The EKS Pod Identity Agent  add-on, which manages AWS credentials for the cluster applications
In addition, the cluster includes a self-managed AWS Load Balancer Controller , that provisions and configures Elastic Load Balancers (Network Load Balancers  for Service resources and Application Load Balancers  for Ingress resources), to expose the cluster applications to traffic.

➤ We can validate that all the applications and controllers are operational (in a Running state) by executing:

kubectl get pods -A

Note that due to the provision process, some Pods may have a non-zero RESTARTS count. As long as these aren't recent (dozens of minutes ago), your cluster is in the desired state.

Cluster Add-ons
The cluster add-ons mentioned above are either DaemonSets (VPC CNI, kube-proxy, and Pod Identity agent) or Deployments (CoreDNS, Karpenter, and Application Load Balancer Controller) and thus their required compute capacity is handled differently.

The DaemonSets will be deployed on every worker node in the cluster, while the rest will be deployed on a specifically configured system-mng managed node group.

The system-mng node group is provisioned along with the cluster and its configuration can be represented, for reference, by the following CloudFormation snippet:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
...
  SystemNodegroup:
    Type: AWS::EKS::Nodegroup
    Properties:
      NodegroupName: system-mng
      ...
      AmiType: AL2023_ARM_64_STANDARD
      NodeRole: !GetAtt NodeRole.Arn
      InstanceTypes:
        - t4g.small
      ScalingConfig:
        MinSize: 0
        DesiredSize: 3
        MaxSize: 3
      Labels:
        role: system-mng
      Taints:
        - Key: role
          Value: system-mng
          Effect: NO_SCHEDULE
      Subnets:
        ...
...
Note the role=system label and the role=system taint in the configuration of the managed node group. These are set to ensure that only the selected software/applications can be scheduled onto this node group's instances.

The relevant add-ons then define a matching toleration and target the node group above using a matching node selector.

➤ View the node group and its configuration in the AWS EKS console  and navigating to the workshop migration cluster, which should look like this:

system-mng Managed Node Group system-mng Managed Node Group Labels system-mng Managed Node Group Taints

➤ Verify the relevant, non-DaemonSet add-ons deployment by executing the following command:

export SYSTEM_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."eks.amazonaws.com/nodegroup" == "system-mng") | ."kubernetes.io/hostname"] | join("\\|")')
export DAEMONSETS_PODS=$(kubectl get ds -n kube-system -o json | jq -r '[.items[].metadata.name] | join ("\\|")')

kubectl get pods --all-namespaces -o wide | grep "${SYSTEM_NODES}" | grep -v "${DAEMONSETS_PODS}"

The output should show at least CoreDNS, Karpenter, and Application Load Balancer Controller, similar to the following:

kube-system   aws-load-balancer-controller-7df858d998-g95mt   1/1     Running   0          7h44m   192.168.174.154   ip-192-168-171-81.us-west-2.compute.internal            <none>           <none>
kube-system   aws-load-balancer-controller-7df858d998-lt62r   1/1     Running   0          7h44m   192.168.126.38    ip-192-168-121-172.us-west-2.compute.internal           <none>           <none>
kube-system   coredns-7d597b58bc-5ftmq                        1/1     Running   0          7h59m   192.168.122.77    ip-192-168-121-172.us-west-2.compute.internal           <none>           <none>
kube-system   coredns-7d597b58bc-rjts4                        1/1     Running   0          7h59m   192.168.122.238   ip-192-168-121-172.us-west-2.compute.internal           <none>           <none>
kube-system   ebs-csi-controller-86cbbcbb67-cltl2             6/6     Running   0          7h46m   192.168.142.236   ip-192-168-150-45.us-west-2.compute.internal            <none>           <none>
kube-system   ebs-csi-controller-86cbbcbb67-rzvks             6/6     Running   0          7h46m   192.168.168.215   ip-192-168-171-81.us-west-2.compute.internal            <none>           <none>
kube-system   karpenter-df586dcf5-4rdrv                       1/1     Running   0          7h44m   192.168.184.184   ip-192-168-171-81.us-west-2.compute.internal            <none>           <none>
kube-system   karpenter-df586dcf5-slc6k                       1/1     Running   0          7h44m   192.168.154.243   ip-192-168-150-45.us-west-2.compute.internal            <none>           <none>
kube-system   metrics-server-76dfb8cb4b-kfkjw                 1/1     Running   0          7h59m   192.168.103.3     ip-192-168-121-172.us-west-2.compute.internal           <none>           <none>
kube-system   metrics-server-76dfb8cb4b-xlj6b                 1/1     Running   0          7h59m   192.168.122.113   ip-192-168-121-172.us-west-2.compute.internal           <none>           <none>
Self-Managed Karpenter Configuration
To allow the self-managed Karpenter to provision instances for our applications' Pods, we need to define at least one NodeClass and one NodePool.

The configuration applied to the cluster can be represented, for reference, by the following partial snippet:

EC2NodeClass:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: apps
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  role: KarpenterNodeRole-${MIGRATION_CLUSTER_NAME}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${MIGRATION_CLUSTER_NAME}
        Name: "*/SubnetPrivate*"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${MIGRATION_CLUSTER_NAME}
NodePool:

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: apps
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  template:
    metadata:
      labels:
        role: apps-karpenter
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: apps
      taints:
        - key: role
          value: apps-karpenter
          effect: NoSchedule
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64, arm64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: node.kubernetes.io/instance-category
          operator: In
          values: [c, m, r]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ['4']
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: [nano, micro, small, medium]
Note the role=apps-karpenter label and the role=apps-karpenter taint in the configuration of the node pool above. These are set, in the same manner as the system-mng node group above, to ensure that only the selected applications can be scheduled using this Karpenter node pool.

Demo Application
To demonstrate the migration process, we will use the demo retail application – a sample application designed to illustrate container-related concepts on AWS.

➤ We can explore the application by executing the command below, Ctrl/Cmd-clicking the URL in the output, and adding a couple of items to the cart:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

The application  contains several components in various languages and frameworks that use pre-built container images for both x86-64 and ARM64 CPU architectures:

Sample Retail Application

The application components are deployed as follows:

Component	Compute Option
Orders	Fargate
Catalog	Managed Node Group
Catalog MySQL database	Managed Node Group
Checkout	Self-Managed Karpenter
Carts	Self-Managed Karpenter
UI	Self-Managed Karpenter
➤ Print out the distribution of instances across all capacity types again by executing the following command:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

This should produce a result similar to the following:

fargate-ip-192-168-100-164.us-west-2.compute.internal  |  us-west-2a  |  FARGATE    |  profile:    apps
fargate-ip-192-168-138-127.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
fargate-ip-192-168-152-183.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
ip-192-168-103-234.us-west-2.compute.internal          |  us-west-2a  |  KARPENTER  |  nodepool:   apps
ip-192-168-131-142.us-west-2.compute.internal          |  us-west-2b  |  KARPENTER  |  nodepool:   apps
ip-192-168-190-197.us-west-2.compute.internal          |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-104-189.us-west-2.compute.internal          |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-132-240.us-west-2.compute.internal          |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-163-151.us-west-2.compute.internal          |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-121-172.us-west-2.compute.internal          |  us-west-2a  |  ON_DEMAND  |  system-mng  
ip-192-168-150-45.us-west-2.compute.internal           |  us-west-2b  |  ON_DEMAND  |  system-mng  
ip-192-168-171-81.us-west-2.compute.internal           |  us-west-2c  |  ON_DEMAND  |  system-mng  
➤ Verify that all the components of the application are deployed as described in the table above:

export DAEMONSETS_PODS=$(kubectl get ds -n kube-system -o json | jq -r '[.items[].metadata.name] | join ("\\|")')
kubectl get pods -n apps -o wide | grep -v "${DAEMONSETS_PODS}"

Now that we have an overall view of the application, we can start the migration process by enabling the EKS Auto Mode for the cluster.

Enabling EKS Auto Mode
Configuring the Cluster | Configuring EKS Auto Mode | Preparing the Application

Configuring the Cluster
Before enabling EKS Auto Mode on our cluster, we need to adjust certain IAM permissions to ensure its proper operation. This includes:

Updating the cluster's IAM role trust policy and permissions
Creating an IAM role for the EKS Auto Mode worker nodes
Let's explore these permissions in more detail in the following sections.

1. Update the Cluster IAM Role
EKS Auto Mode includes several Kubernetes capabilities as core components. These components, that would otherwise have to be managed as add-ons or self-managed controllers, include built-in support for Pod IP address assignments, Pod network policies, local DNS services, GPU plug-ins, health checkers, and EBS CSI storage.

To manage these components and ensure their function, EKS Auto Mode requires additional permissions, which are now a part of the cluster IAM role.

As described in the EKS Auto Mode documentation , the following IAM policies should be added to the cluster role, in addition to the existing AmazonEKSClusterPolicy:

AmazonEKSComputePolicy 
AmazonEKSBlockStoragePolicy 
AmazonEKSNetworkingPolicy 
AmazonEKSLoadBalancingPolicy 
In addition, EKS Auto Mode requires the sts:TagSession action to be added to the cluster IAM role's trust policy (you can view its current state in the IAM console).

➤ Create a trust policy JSON file:

cat << EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "eks.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession"
            ]
        }
    ]
}
EOF

➤ Update the cluster IAM role and remove the trust-policy.json file:

aws iam update-assume-role-policy \
  --role-name ${MIGRATION_CLUSTER_ROLE_NAME} \
  --policy-document file://trust-policy.json

rm trust-policy.json

➤ Add the required permissions to the cluster IAM role by executing:

for POLICY_ARN in \
  "arn:aws:iam::aws:policy/AmazonEKSComputePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSBlockStoragePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSNetworkingPolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSLoadBalancingPolicy"
do
  echo "Attaching policy ${POLICY_ARN} to IAM role ${MIGRATION_CLUSTER_ROLE_NAME}..."
  aws iam attach-role-policy --role-name ${MIGRATION_CLUSTER_ROLE_NAME} \
    --policy-arn ${POLICY_ARN}
done

➤ Verify that the IAM role has been updated properly:

aws iam get-role --role-name ${MIGRATION_CLUSTER_ROLE_NAME} | \
  jq -r '.Role.AssumeRolePolicyDocument.Statement[].Action[]'

aws iam list-attached-role-policies --role-name ${MIGRATION_CLUSTER_ROLE_NAME} | \
  jq -r '.AttachedPolicies[].PolicyName'

The output should include (note the AmazonEKSClusterPolicy that the role already had attached before):

sts:AssumeRole
sts:TagSession
AmazonEKSClusterPolicy
AmazonEKSNetworkingPolicy
AmazonEKSComputePolicy
AmazonEKSBlockStoragePolicy
AmazonEKSLoadBalancingPolicy
2. Create the Node IAM Role
A Node IAM role contains the minimal permissions required for EKS components (kubelet and Pod Identity agent) to perform their function.

➤ Create a trust policy JSON file:

cat << EOF > trust-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": [
                "sts:AssumeRole"
            ]
        }
    ]
}
EOF

➤ Create the Node IAM role and remove the trust-policy.json file:

aws iam create-role \
  --role-name ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role \
  --assume-role-policy-document file://trust-policy.json

rm trust-policy.json

➤ Attach the required policies:

for POLICY_ARN in \
  "arn:aws:iam::aws:policy/AmazonEKSWorkerNodeMinimalPolicy" \
  "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"
do
  echo "Attaching policy ${POLICY_ARN} to IAM role ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role..."
  aws iam attach-role-policy --role-name ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role \
    --policy-arn ${POLICY_ARN}
done

➤ Verify the created (or existing) role, its trust policy, and attached managed policies:

aws iam get-role \
  --role-name ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role | \
  jq -r '.Role.AssumeRolePolicyDocument.Statement[].Action'

aws iam list-attached-role-policies \
  --role-name ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role | \
  jq -r '.AttachedPolicies[].PolicyName'

The command above should produce the following output:

sts:AssumeRole
AmazonEKSWorkerNodeMinimalPolicy
AmazonEC2ContainerRegistryPullOnly
We can now enable EKS Auto Mode.

3. Enable Amazon EKS Auto Mode
➤ Define the node role ARN as an environment variable:

export NODE_ROLE_ARN=$(aws iam get-role \
  --role-name ${MIGRATION_CLUSTER_NAME}-auto-mode-node-role \
  --query "Role.Arn" --output text)

➤ Execute the following command to enable EKS Auto Mode (and verify that there are no errors in the output):

aws eks update-cluster-config \
  --region ${AWS_REGION} \
  --name ${MIGRATION_CLUSTER_NAME} \
  --compute-config "{\"nodeRoleArn\": \"${NODE_ROLE_ARN}\", \"nodePools\": [\"system\"], \"enabled\": true}" \
  --kubernetes-network-config '{"elasticLoadBalancing":{"enabled": true}}' \
  --storage-config '{"blockStorage":{"enabled": true}}'

➤ After 5 - 10 seconds, execute the following command to wait for the cluster EKS Auto Mode to become enabled:

aws eks wait cluster-active --name ${MIGRATION_CLUSTER_NAME}

You can also verify that the process of enabling EKS Auto Mode has started by navigating to the AWS EKS console , selecting the migration cluster, and ensuring that the corresponding panel looks like this (note the spinning icon on the Manage button):

Enabling EKS Auto Mode progress

After a couple of minutes, the cluster should enable EKS Auto Mode:

Enabled EKS Auto Mode

Before proceeding with the migration, let's make sure the Auto Mode autoscaling configuration exists in the cluster.

➤ Verify that all node pools (including those managed by the self-managed Karpenter) and node classes are fully operational (READY=True):

kubectl get nodepool,nodeclass,ec2nodeclass

You should see an output similar to the following, which shows the original self-managed Karpenter NodePool and EC2NodeClass, as well as the EKS Auto Mode NodePool and (a new CRD) NodeClass:

NAME                           NODECLASS   NODES   READY   AGE
nodepool.karpenter.sh/apps     apps        8       True    89m
nodepool.karpenter.sh/system   default     0       True    8m47s

NAME                                  ROLE                                    READY   AGE
nodeclass.eks.amazonaws.com/default   migration-cluster-auto-mode-node-role   True    8m47s

NAME                                  READY   AGE
ec2nodeclass.karpenter.k8s.aws/apps   True    89m
➤ Extract (if you already closed its tab in the browser) the application URL:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

➤ Verify that the application is operational by Ctrl/Cmd-clicking the URL in the output (or refreshing the tab).

We are now ready to start the migration process of our applications to EKS Auto Mode-managed instances.

Configuring EKS Auto Mode
EKS Auto Mode manages most infrastructure components automatically, while allowing customization to better fit the needs of the workloads.

EKS Auto Mode includes two built-in NodePools (and a NodeClass) that define how the cluster compute capacity is provisioned, out of the box:

The system node pool, intended for cluster-critical applications
The general-purpose node pool
While the general-purpose would work for most general applications' needs and help customers to get started (as seen in the Getting Started section of one of the previous modules), for the purposes of the migration, we will use a different node pool.

Note that we opted out of creating the general-purpose node pool in the update-cluster-config command above.

We will use the configuration of the apps node pool we already have as a starting point, with some minor adjustments, to create a new custom node pool for EKS Auto Mode.

Note that EKS Auto Mode doesn't allow changing the general-purpose node pool and the corresponding default node class. Hence, it is recommended to analyze the Kubernetes scheduling constraints in use such as labels, taints, and tolerations, affinity, and anti-affinity rules, and create EKS Auto Mode node pools accordingly.

Create a Custom EKS Auto Mode Configuration (NodeClass and NodePool)
➤ Create the apps-auto-mode-np.yml file:

cat << EOF > apps-auto-mode-np.yml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: apps-auto-mode
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  template:
    metadata:
      labels:
        role: apps-auto-mode
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      taints:
        - key: role
          value: apps-auto-mode
          effect: NoSchedule
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64, arm64]
        - key: kubernetes.io/os
          operator: In
          values: [linux]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: eks.amazonaws.com/instance-category
          operator: In
          values: [c, m, r]
        - key: eks.amazonaws.com/instance-generation
          operator: Gt
          values: ["4"]
EOF

To adjust the apps NodePool to EKS Auto Mode, we’ve introduced the following changes:

Renamed the node pool to apps-auto-mode
Changed the node class reference to the EKS Auto Mode default NodeClass
Renamed the label and the taint values to apps-auto-mode
Removed the karpenter.k8s.aws/instance-generation as it is restricted by EKS Auto Mode
➤ Deploy the node pool:

kubectl apply -f apps-auto-mode-np.yml

➤ Verify that all node pools and node classes are fully operational (READY=True):

kubectl get nodepool,nodeclass,ec2nodeclass

Note that at this point no instances are registered with any of the node pools:

NAME                                   NODECLASS   NODES   READY   AGE
nodepool.karpenter.sh/apps             apps        8       True    92m
nodepool.karpenter.sh/apps-auto-mode   default     0       True    56s
nodepool.karpenter.sh/system           default     0       True    12m

NAME                                  ROLE                                    READY   AGE
nodeclass.eks.amazonaws.com/default   migration-cluster-auto-mode-node-role   True    12m

NAME                                  READY   AGE
ec2nodeclass.karpenter.k8s.aws/apps   True    92m
Preparing the Application
Before applying any changes, we need to ensure that the components in the cluster can withstand them with minimal impact. If we were to simply delete the nodes, all Pods associated with the nodes would be terminated and need to be re-scheduled at the same time, causing disruption to their services.

In Kubernetes, there are several ways of controlling disruptions  and the decision to apply them depends on the type of disruption  we are about to introduce.

These ways are PodDisruptionBudgets  and Deployment strategy .

Since our application components are deployed using Helm, which updates the Deployment resource, we will use a Deployment strategy, which is already included in our application components and looks like this:

...
strategy:
  rollingUpdate:
    maxUnavailable: 1
  type: RollingUpdate...
Note that while the above strategy suits our needs in the context of the workshop, for your applications, you should create strategies that match your applications' requirements.

In addition to ensuring a controlled migration process above, we also need to migrate the network resources created by/for the application components.

Our retail store demo application defines an Ingress  resource to expose its components to external traffic.

This Ingress is then handled by the self-managed AWS Load Balancer Controller to provision the Application Load Balancer above and define rules according to the Ingress definitions.

We will discuss migrating the network components in greater detail in a dedicated section further in the workshop, but the process will rely on DNS migration from the current setup to a network path we will create by leveraging the EKS Auto Mode load balancing capabilities.

With EKS Auto Mode enabled and the application prepared, we can move to migrating the application components, compute option by compute option.


Migrating from AWS Fargate
Migrate the orders Component

In this section of the module, we will demonstrate how to migrate applications deployed in Fargate to EKS Auto Mode using the orders component of our retail store application.

We will update the component to migrate to EKS Auto Mode by adding the relevant node selector and tolerations to align with the requirements defined in the apps-auto-mode EKS Auto Mode node pool.

Let's make sure we check the application health and availability by visiting the application load balancer URL from a browser and adding a few items to the cart.

We can access the application UI by executing the following command to extract the Application Load Balancer DNS name and Ctrl/Cmd-clicking on the URL in the output:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

Migrate the orders Component
➤ Observe the current state of the orders component:

kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-orders -o wide

The output should show, similar to the below, that the orders component is currently hosted exclusively on Fargate nodes.

NAME                                       READY   STATUS    RESTARTS   AGE     IP            NODE                                                   NOMINATED NODE   READINESS GATES
retail-store-app-orders-5d5d9d9dfc-7wc6s   1/1     Running   0          3h29m   10.0.74.111   fargate-ip-10-0-74-111.eu-central-1.compute.internal   <none>           <none>
retail-store-app-orders-5d5d9d9dfc-fkks2   1/1     Running   0          3h24m   10.0.85.172   fargate-ip-10-0-85-172.eu-central-1.compute.internal   <none>           <none>
retail-store-app-orders-5d5d9d9dfc-mhw9h   1/1     Running   0          3h24m   10.0.49.132   fargate-ip-10-0-49-132.eu-central-1.compute.internal   <none>           <none>
➤ You can review the distribution of instances across all capacity types by executing the following command:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The command output should show the Fargate instances that host the orders component above:

fargate-ip-192-168-116-56.us-west-2.compute.internal   |  us-west-2c  |  FARGATE    |  profile:    apps
fargate-ip-192-168-180-168.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
fargate-ip-192-168-191-115.us-west-2.compute.internal  |  us-west-2b  |  FARGATE    |  profile:    apps
ip-192-168-103-128.us-west-2.compute.internal          |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-114-228.us-west-2.compute.internal          |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-127-58.us-west-2.compute.internal           |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-175-162.us-west-2.compute.internal          |  us-west-2b  |  KARPENTER  |  nodepool:   apps
ip-192-168-28-67.us-west-2.compute.internal            |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-51-140.us-west-2.compute.internal           |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-83-58.us-west-2.compute.internal            |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-26-158.us-west-2.compute.internal           |  us-west-2c  |  ON_DEMAND  |  system-mng  
ip-192-168-33-46.us-west-2.compute.internal            |  us-west-2a  |  ON_DEMAND  |  system-mng  
ip-192-168-86-255.us-west-2.compute.internal           |  us-west-2b  |  ON_DEMAND  |  system-mng  
➤ Create the orders-values.yml file for the orders component (see here  for the whole values.yml file):

cat << EOF > orders-values.yml
replicaCount: 3

podAnnotations:
  eks.amazonaws.com/compute-type: ec2

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule

topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 3
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/instance: retail-store-app-orders
EOF

To ensure that the orders components are hosted on instances provisioned through the EKS Auto Mode node pool we defined in the previous section, we've added the corresponding node selector and tolerations.

Note that we added a Pod annotation: eks.amazonaws.com/compute-type above to ensure that Pods currently deployed on Fargate instances would gradually migrate to EKS Auto Mode.

Had we created the node selector only, without specifying the annotation, the Pods would be terminated and remained in the Pending state, since the apps Fargate profile instances do not have the corresponding label.

Alternatively, had we deleted the apps Fargate profile, that would cause all orders Pods to be terminated and scheduled at the same time, causing disruption to the application's services.

➤ In a separate VS Code terminal, execute the following command to observe the migration process:

kubectl get pods -n apps -l app.kubernetes.io/name=orders -w

➤ Update the orders component:

1
2
3
4
5
helm upgrade retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values orders-values.yml \
  --wait

➤ Observe that the application remains operational, while the component is being migrated, by navigating to the application UI, Ctrl/Cmd-click the URL in the output, and adding a couple of items to the cart:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

➤ After a couple of minutes, verify that all orders Pods are scheduled on the apps-auto-mode node pool by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-orders -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

The expected output should be similar to the following, where the instance name being of the form i-xxxxxxxxxxxxxxxxx indicates that it belongs to EKS Auto Mode:

retail-store-app-orders-55c98b46d4-cd68l   1/1     Running   0          5m39s   192.168.8.160    i-0a075bf0169b36e2c
retail-store-app-orders-55c98b46d4-cn6mm   1/1     Running   0          6m21s   192.168.74.64    i-0b694975e1e8ac9c7
retail-store-app-orders-55c98b46d4-fzxt6   1/1     Running   0          2m18s   192.168.42.224   i-001a27779836de720
➤ We can, once more, review the distribution of instances across all capacity types by executing the following command and observing the change (removal of Fargate nodes and a new type of nodes provisioned via EKS Auto Mode apps-auto-mode node pool)

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The output should show instances provisioned through new EKS Auto Mode nodepool and no Fargate instances in the cluster:

ip-192-168-153-77.us-west-2.compute.internal   |  us-west-2b  |  KARPENTER  |  nodepool:   apps
ip-192-168-165-109.us-west-2.compute.internal  |  us-west-2c  |  KARPENTER  |  nodepool:   apps
ip-192-168-97-157.us-west-2.compute.internal   |  us-west-2a  |  KARPENTER  |  nodepool:   apps
i-001a27779836de720                            |  us-west-2b  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0a075bf0169b36e2c                            |  us-west-2a  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0b694975e1e8ac9c7                            |  us-west-2c  |  KARPENTER  |  nodepool:   apps-auto-mode
ip-192-168-136-11.us-west-2.compute.internal   |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-171-62.us-west-2.compute.internal   |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-98-145.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-126-11.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  system-mng  
ip-192-168-151-213.us-west-2.compute.internal  |  us-west-2b  |  ON_DEMAND  |  system-mng  
ip-192-168-184-34.us-west-2.compute.internal   |  us-west-2c  |  ON_DEMAND  |  system-mng  

Migrating from EKS Managed Node Groups
Migrate the Catalog Component

In this section of the module, we will migrate the catalog component and its MySQL database that are deployed onto the apps-mng managed node group.

We will update these components to migrate to EKS Auto Mode by adding the relevant node selector and tolerations to align with the requirements defined in the apps-auto-mode EKS Auto Mode node pool.

Migrate the catalog Component
For this assignment, we will only migrate the deployment part of the component and leave the StatefulSet as it is now, with no impact on the application. Migrating EBS-based stateful applications, which MySQL is, requires a separate process, which we'll explore in the later assignment called "Migrating Stateful Applications".

➤ Create the catalog-values.yml file for the catalog component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
cat << EOF > catalog-values.yml
replicaCount: 3

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule

app:
  persistence:
    provider: mysql
    endpoint: ""
    database: "catalog"

    secret:
      create: true
      name: catalog-db
      username: catalog
      password: "mysqlcatalog123"

mysql:
  create: true
  nodeSelector:
    role: apps-mng
  tolerations:
    - key: role
      value: apps-mng
      operator: Equal
      effect: NoSchedule

  persistentVolume:
    enabled: true
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    storageClass: "gp3"
EOF

➤ In a separate VS Code terminal, execute the following command to observe the migration process:

kubectl get pods -n apps -l app.kubernetes.io/name=catalog -w

➤ Verify that all catalog component pods are still on the original managed node groups:

kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-catalog -o wide

➤ Review the distribution of instances across all capacity types by executing the following command and observing the change (note the nodes where the catalog is running):

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

➤ Now, let's update the catalog component:

1
2
3
4
5
6
helm upgrade retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values catalog-values.yml \
  --reuse-values \
  --wait

➤ After a couple of minutes, verify that all catalog Pods are scheduled on the apps-auto-mode nodes (again, identified by their i-xxxxxxxxxxxxxxxxx name) by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-catalog -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

➤ Verify that the catalog component's MySQL StatefulSet pods are still on the original managed node groups:

kubectl get pods -n apps -l app.kubernetes.io/name=retail-store-app-catalog -o wide

➤ As before, review the distribution of instances across all capacity types by executing the following command and observing the change (removal of Fargate nodes and a new type of nodes provisioned via EKS Auto Mode apps-auto-mode node pool):

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The output should show instances provisioned through the new EKS Auto Mode NodePool and no Fargate instances in the cluster:

ip-192-168-109-141.us-west-2.compute.internal  |  us-west-2a  |  KARPENTER  |  nodepool:   apps
ip-192-168-144-11.us-west-2.compute.internal   |  us-west-2b  |  KARPENTER  |  nodepool:   apps
ip-192-168-167-219.us-west-2.compute.internal  |  us-west-2c  |  KARPENTER  |  nodepool:   apps
i-085c7fe5654a651c8                            |  us-west-2a  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0ce7c07eb76652c22                            |  us-west-2c  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0d26a453b47d25d9f                            |  us-west-2b  |  KARPENTER  |  nodepool:   apps-auto-mode
ip-192-168-112-81.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-150-83.us-west-2.compute.internal   |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-184-40.us-west-2.compute.internal   |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-133-200.us-west-2.compute.internal  |  us-west-2b  |  ON_DEMAND  |  system-mng  
ip-192-168-171-235.us-west-2.compute.internal  |  us-west-2c  |  ON_DEMAND  |  system-mng  
ip-192-168-99-105.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  system-mng 

Migrating from a Self-Managed Karpenter
Migrate the Checkout Component | Migrate the Carts Component | Migrate the UI Component

In this section of the module, we will demonstrate how to migrate applications from a self-managed Karpenter to EKS Auto Mode.

In the current setup, the self-managed Karpenter controller handles the compute requirements for the checkout, carts, and UI components of the retail store application.

We will update these components to migrate to EKS Auto Mode by adding the relevant node selector and tolerations to align with the requirements defined in the apps-auto-mode EKS Auto Mode node pool.

Migrate the checkout Component
➤ Create the checkout-values.yml file for the checkout component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
cat << EOF > checkout-values.yml
replicaCount: 3

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule

redis:
  nodeSelector:
    role: apps-auto-mode
  tolerations:
    - key: role
      value: apps-auto-mode
      operator: Equal
      effect: NoSchedule
EOF

➤ In a separate IDE terminal, execute the following command to observe the migration process:

kubectl get pods -n apps -l app.kubernetes.io/name=checkout -w

➤ Update the checkout component:

1
2
3
4
5
helm upgrade retail-store-app-checkout oci://public.ecr.aws/aws-containers/retail-store-sample-checkout-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values checkout-values.yml \
  --wait

➤ After a couple of minutes, we can verify that all checkout Pods are scheduled on the apps-auto-mode nodes by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-checkout -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

Migrate the carts Component
➤ Create the carts-values.yml file for the carts component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
cat << EOF > carts-values.yml
replicaCount: 3

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule

dynamodb:
  nodeSelector:
    role: apps-auto-mode
  tolerations:
    - key: role
      value: apps-auto-mode
      operator: Equal
      effect: NoSchedule
EOF

➤ In a separate IDE terminal, execute the following command to observe the migration process:

kubectl get pods -n apps -l app.kubernetes.io/name=carts -w

➤ Now, let's update the carts component:

1
2
3
4
5
helm upgrade retail-store-app-carts oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values carts-values.yml \
  --wait

➤ After a couple of minutes, verify that all carts Pods are scheduled on the apps-auto-mode nodes by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-carts -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

Migrate the UI Component
➤ Create the ui-values.yml file for the UI component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
cat << EOF > ui-values.yml
replicaCount: 3

app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule
EOF

➤ In a separate IDE terminal, execute the following command to observe the migration process:

kubectl get pods -n apps -l app.kubernetes.io/name=ui -w

➤ Update the UI component:

1
2
3
4
5
6
helm upgrade retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values ui-values.yml \
  --reuse-values \
  --wait

➤ After a couple of minutes, verify that all ui Pods are scheduled on the apps-auto-mode nodes by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -l app.kubernetes.io/instance=retail-store-app-ui -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

➤ View all the Pods (excluding DaemonSets) and their nodes to confirm the migration:

export DAEMONSETS_PODS=$(kubectl get ds -n kube-system -o json | jq -r '[.items[].metadata.name] | join ("\\|")')
kubectl get pods -A -o wide | grep -v "${DAEMONSETS_PODS}"

We can omit DaemonSets to reduce visual clutter, since they do not impact the compute distribution.

The expected output should be similar to the following (omitting the kube-system namespace components for brevity):

NAMESPACE     NAME                                            READY   STATUS    RESTARTS   AGE     IP                NODE                                            NOMINATED NODE   READINESS GATES
apps          retail-store-app-carts-7f8d59668-68nsc          1/1     Running   0          4m15s   192.168.46.178    i-0d26a453b47d25d9f                             <none>           <none>
apps          retail-store-app-carts-7f8d59668-pbwzm          1/1     Running   0          4m15s   192.168.8.242     i-085c7fe5654a651c8                             <none>           <none>
apps          retail-store-app-carts-7f8d59668-pgvjm          1/1     Running   0          3m55s   192.168.74.149    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-catalog-59d4c7ddcf-47zdb       1/1     Running   0          11m     192.168.8.240     i-085c7fe5654a651c8                             <none>           <none>
apps          retail-store-app-catalog-59d4c7ddcf-99tgc       1/1     Running   0          11m     192.168.46.176    i-0d26a453b47d25d9f                             <none>           <none>
apps          retail-store-app-catalog-59d4c7ddcf-n4l6h       1/1     Running   0          11m     192.168.74.147    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-catalog-mysql-0                1/1     Running   0          2d6h    192.168.119.154   ip-192-168-112-81.us-west-2.compute.internal    <none>           <none>
apps          retail-store-app-checkout-6d9f5b4d4c-6t9kf      1/1     Running   0          5m21s   192.168.8.241     i-085c7fe5654a651c8                             <none>           <none>
apps          retail-store-app-checkout-6d9f5b4d4c-ckczm      1/1     Running   0          4m58s   192.168.74.148    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-checkout-6d9f5b4d4c-rd769      1/1     Running   0          5m21s   192.168.46.177    i-0d26a453b47d25d9f                             <none>           <none>
apps          retail-store-app-orders-7f8bfccb58-5vkh7        1/1     Running   0          14m     192.168.74.144    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-orders-7f8bfccb58-9gw88        1/1     Running   0          13m     192.168.74.146    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-orders-7f8bfccb58-b72kt        1/1     Running   0          14m     192.168.74.145    i-0ce7c07eb76652c22                             <none>           <none>
apps          retail-store-app-ui-fb8d55cc9-65vq9             1/1     Running   0          49s     192.168.46.179    i-0d26a453b47d25d9f                             <none>           <none>
apps          retail-store-app-ui-fb8d55cc9-vpjkw             1/1     Running   0          49s     192.168.8.243     i-085c7fe5654a651c8                             <none>           <none>
apps          retail-store-app-ui-fb8d55cc9-vr85v             1/1     Running   0          32s     192.168.74.150    i-0ce7c07eb76652c22                             <none>           <none>
...
You can see that all other application components' Pods in the apps namespace are running on EKS Auto Mode instances, while the MySQL and cluster operation software (controllers and add-ons) are not.

➤ View the distribution of instances across capacity types:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The result should be similar to the following:

i-085c7fe5654a651c8                            |  us-west-2a  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0ce7c07eb76652c22                            |  us-west-2c  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0d26a453b47d25d9f                            |  us-west-2b  |  KARPENTER  |  nodepool:   apps-auto-mode
...
ip-192-168-112-81.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-150-83.us-west-2.compute.internal   |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-184-40.us-west-2.compute.internal   |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-133-200.us-west-2.compute.internal  |  us-west-2b  |  ON_DEMAND  |  system-mng  
ip-192-168-171-235.us-west-2.compute.internal  |  us-west-2c  |  ON_DEMAND  |  system-mng  
ip-192-168-99-105.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  system-mng  
Observing the `apps` node pool
Note that you may continue to see the apps node pool with its now empty nodes, while Karpenter terminates them.

We can verify (by running kubectl describe node... commands for system-mng and apps-mng node group nodes) that the only non-DaemonSet pods that remain on them are the operational software (like self-managed Karpenter) and the catalog component's MySQL pod.

We now successfully migrated all the application components pods (again, excluding catalog component's MySQL database) to EKS Auto Mode.

Migrating Networking Resources
Create New Load Balancers | Migrate to the New Load Balancers

In this section, we will demonstrate how to migrate networking resources, specifically AWS Application Load Balancer (ALB) and Network Load Balancer (NLB).

When using the AWS Load Balancer Controller, as in our setup, creating a Kubernetes Ingress resource automatically provisions an AWS ALB, while creating a Kubernetes Service resource of type LoadBalancer provisions an AWS NLB for application traffic distribution.

Create New Load Balancers
In our workshop setup, the UI application is configured with a Kubernetes Ingress resource and is accessible through an AWS ALB.

Access the UI
You can access the application UI by executing the following command to extract the ALB DNS name and Ctrl/Cmd-clicking on the URL in the output:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

When migrating applications that use LoadBalancers to EKS Auto Mode, we need to create new load balancers, as existing ones, managed by the self-managed AWS Load Balancer controller, cannot be migrated to EKS Auto Mode.

See here  for additional information on migrating resources in existing Amazon EKS clusters.

In the UI component Helm implementation, we have defined Ingress resources as an array , enabling us to create a new Ingress resource while maintaining the existing Ingress-created ALB during migration.

➤ View the application's Ingress resources:

kubectl get ingress -n apps

If you're not using Helm, we recommend creating a new Ingress resource with a different name that uses the Ingress className created specifically for EKS Auto Mode.

So, before proceeding with the migration, we will create a new IngressClass.

This setup ensures that the necessary infrastructure configurations are in place to support ingress requirements for applications deployed later. Typically, this is a step performed by the platform team once after the cluster is created.

➤ Define the new IngressClass:

cat << EOF > ingress-class.yml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-mode-alb
spec:
  scheme: internet-facing
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-mode-alb
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-mode-alb
EOF

➤ Now let's deploy the IngressClass:

kubectl apply -f ingress-class.yml

With EKS Auto Mode enabled and the application prepared, we can move to migrating the application components, compute option by compute option.

In the following Helm configuration, we are configuring a new Ingress resource by using a new IngressClass with the className: eks-auto-alb for the second Ingress resource definition.

➤ Create the ui-ingress-values.yml file for the UI component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
cat << EOF > ui-ingress-values.yml
ingresses:
  - enabled: true
    className: alb
    name: main
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
  - enabled: true
    className: eks-auto-mode-alb
    name: main-auto-mode
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
EOF

Note that we need to define both the old and the new Ingress resources to avoid overriding the original configuration.

➤ Update the UI component:

1
2
3
4
5
6
helm upgrade retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values ui-ingress-values.yml \
  --reuse-values \
  --wait

➤ Verify that there are now two Ingress resources:

kubectl get ingress -n apps

➤ After a couple of minutes, let's verify that we can access the application UI via the new ALB by executing the following command to extract the DNS name and Ctrl/Cmd-clicking on the URL in the output:

export AUTO_MODE_ALB_URL=$(kubectl get ingress -n apps retail-store-app-ui-main-auto-mode -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$AUTO_MODE_ALB_URL"'`].LoadBalancerArn' --output text)
echo "The shared ALB is available at: http://$AUTO_MODE_ALB_URL"

➤ Also let's verify that we can access the application UI via the old ALB by executing the following command to extract the DNS name and Ctrl/Cmd-clicking on the URL in the output:

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

Migrate to the New Load Balancers
To ensure a successful migration, we should follow several best practices:

Schedule migrations during low-traffic periods and implement a DNS-based migration strategy for zero downtime.

EKS Auto Mode maintains compatibility with external-dns , which automatically registers new load balancers with Route 53. If we're not using external-dns, we'll need to update DNS entries either manually or through automation.

During the migration period, it is essential to maintain both the old and the new load balancers while continuously monitoring performance metrics.

Have a comprehensive rollback plan ready for any unexpected issues.

Once testing confirms a successful migration, we can remove the old load balancer by deleting its Ingress resource. In this example, we'll execute a helm upgrade to update the ingress array, removing the old ingress entry.

➤ Create the ui-single-ingress-values.yml file:

1
2
3
4
5
6
7
8
9
cat << EOF > ui-single-ingress-values.yml
ingresses:
  - enabled: true
    className: eks-auto-mode-alb
    name: main-auto-mode
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
EOF

Note that the array above removes the second Ingress resource by overriding the entire ingresses array.

➤ Update the UI component:

1
2
3
4
5
6
helm upgrade retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values ui-single-ingress-values.yml \
  --reuse-values \
  --wait

➤ After a couple of minutes, let's verify that only one Ingress resource remains:

kubectl get ingress -n apps

➤ Let's verify that we can access the application UI via the new ALB by executing the following command to extract the DNS name and Ctrl/Cmd-clicking on the URL in the output:

kubectl get ingress -n apps retail-store-app-ui-main-auto-mode \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

➤ Let's verify that we can no longer retrieve the old ALB (expected to receive error):

kubectl get ingress -n apps retail-store-app-ui-main \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

Note that SSL certificates can be shared between your old and new load balancers. However, ensure that your certificate is configured with appropriate Subject Alternative Names (SANs) that match all domains served by both ALBs, particularly if you're changing domain names during migration.

Server Name Indication (SNI) enables this functionality, allowing the ALB to select the correct certificate based on the client's requested hostname.

We have now migrated almost all of our retail store application, with the catalog component's MySQL database as the only thing remaining.

Migrating Stateful Components
Overview | Migrate the catalog component's MySQL database

Overview
In our application, we have multiple stateful components:

Orders PostgreSQL database
Catalog MySQL database
Checkout Redis in-memory
Carts in-cluster DynamoDB
Sample Retail Application

Applications deployed in Amazon EKS can use several AWS storage services, including Amazon S3 , Amazon EBS , and Amazon EFS , through an appropriate CSI driver .

For simplicity and brevity during this workshop, most of the stateful sub-components (Redis, PostgreSQL, and DynamoDB) use their pods' file system (via the emptyDir ephemeral volumes) or memory. Only the catalog component's MySQL database relies on the Amazon EBS CSI driver  to provision an EBS volume to host its filesystem.

Specifically, it uses dynamic provisioning  to trigger the automatic provisioning and attachment of an EBS volume when the MySQL StatefulSet is created.

Workshop stateful components
➤ We can specifically explore these components by executing the following command:

kubectl get sts -n apps -o wide

➤ We can verify that executing the following command and verifying that only a single pod, the MySQL StatefulSet, still runs on non-EKS Auto Mode compute:

kubectl get pods -n apps -o wide

Catalog component stateful components
➤ View the catalog component's MySQL StatefulSet pods:

kubectl get sts -n apps -l app.kubernetes.io/component=mysql

➤ View the underlying PersistentVolume and PersistentVolumeClaim:

kubectl get -n apps pv,pvc

➤ View the StorageClasses:

kubectl get sc

You can see the connection between these components:

Dynamic provision

Migration challenges
The main challenges in handling the migration of EBS-based stateful components that rely on dynamic provisioning can be summarized as follows:

It is not possible to transfer the “ownership” of a PersistentVolume, as the spec.persistentvolumesource (which contains the reference to the handling driver) is immutable after creation.

The StorageClass attribute in a PersistentVolumeClaim object is immutable and cannot be updated.

It is not possible to attach both EBS CSI and Auto Mode (empty) volumes to the same node and mount them into the same Pod to perform an application-level migration of data.

Migrate the catalog component's MySQL database
We will demonstrate the migration of these components using the catalog component's MySQL as an example, but any stateful application that uses the EBS CSI driver in the same manner as above can follow the same process we are going to outline and execute below.

Additionally, for simplicity, we will call OSS EBS CSI driver-managed (and adjacent) resources “EBS CSI resources” and the Amazon EKS Auto Mode managed (and adjacent) resources “Auto Mode resources”.

In light of the challenges listed above, the migration will introduce a short downtime to the application – in this case, the catalog component's MySQL database.

For that purpose, we have developed a migration tool  that automates the process to reduce operational overhead and minimize the incurred downtime.

ATTENTION
Note that the migration process requires deleting the existing PersistentVolumeClaim/PersistentVolume and re-creating them with the new StorageClass. You must validate this process in an identical non-production environment first.

Prerequisites
The migration process validates and requires that:

The EBS volume behind the PersistentVolume must be unattached from any EC2 instances. This means scaling down the StatefulSet or Deployment to allow the volume to detach.

The new StorageClass (that belongs to EKS Auto Mode) must have a volumeBindingMode of WaitForFirstConsumer to prevent the immediate creation of a PersistentVolume.

The existing PersistentVolume must have a reclaim policy of Retain to ensure that the EBS volume remains when the PersistentVolume is deleted.

The calling process of the tool needs the appropriate Kubernetes permissions to Create/Delete the PersistentVolumeClaim and PersistentVolume (see here  for more information).

The calling process of the tool needs the appropriate AWS IAM permissions to call DescribeVolume, CreateTags on the EBS volume, and optionally, but recommended, CreateSnapshot.

➤ Execute the following:

export MYSQL_STS_NAME=retail-store-app-catalog-mysql
export MYSQL_POD_NAME=${MYSQL_STS_NAME}-0
export ORIGINAL_PVC_NAME=data-${MYSQL_POD_NAME}
echo ${ORIGINAL_PVC_NAME}

Download and install the tool
➤ Download the tool:

curl -sSL -o eks-auto-mode-ebs-migration-tool https://github.com/awslabs/eks-auto-mode-ebs-migration-tool/releases/download/v0.3.1/eks-auto-mode-ebs-migration-tool_Linux_x86_64

chmod +x eks-auto-mode-ebs-migration-tool

./eks-auto-mode-ebs-migration-tool

➤ Create a new EKS Auto Mode StorageClass manifest:

cat << EOF > am-storage-class.yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp3-auto
provisioner: ebs.csi.eks.amazonaws.com
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  fsType: ext4
EOF

➤ Create the storage class:

kubectl apply -f am-storage-class.yml

➤ Execute the tool in the (default) dry-run mode to assess the changes planned to be done to the relevant objects and resources:

./eks-auto-mode-ebs-migration-tool \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --pvc-name ${ORIGINAL_PVC_NAME} \
  --namespace apps \
  --storageclass gp3-auto

As you can see, the EBS volume is still attached to the EC2 instance on which the catalog component's StatefulSet Pod is running, so it can't yet be migrated, as mentioned in the prerequisites.

E0606 09:50:35.341373 296117 main.go:135] "Precondition checks failed" err="can't migrate volume vol-07698e9d8d3917a0b that is still attached to an instance"
➤ For the sake of completeness, verify that it is the same EC2 instance by extracting its name from the Pod and via the volume:

export ORIGINAL_PV_NAME=$(kubectl get -n apps pvc ${ORIGINAL_PVC_NAME} -o jsonpath='{.spec.volumeName}')
export VOLUME_ID=$(kubectl get -n apps pv ${ORIGINAL_PV_NAME} -o jsonpath='{.spec.csi.volumeHandle}')

# Get the node name where the MySQL Pod runs on
kubectl get pods -n apps ${MYSQL_POD_NAME} -o json | jq -r '.spec.nodeName'

# Get the node name that the EBS Volume of the PVC is attached to
aws ec2 describe-instances --instance-ids $(aws ec2 describe-volumes --volume-ids ${VOLUME_ID} --output text --query='Volumes[].Attachments[].InstanceId') \
  --output text \
  --query='Reservations[].Instances[].PrivateDnsName'

The expected output is the same node name twice (one from the kubectl command, and one from the aws cli command):

ip-192-168-104-189.us-west-2.compute.internal
ip-192-168-104-189.us-west-2.compute.internal
Before scaling down the catalog component's StatefulSet to allow the volume to detach, consider other factors that may interfere with the number of replicas, most notably active application autoscaling tools like Horizontal Pod Autoscaling (HPA)  or KEDA .

It is recommended that you adjust their configuration (or disable them, if possible) to prevent scaling events during the EBS volume migration.

Our catalog component's StatefulSet doesn't have an HPA attached, so we can safely proceed with scaling it down.

➤ Execute the following command:

kubectl scale statefulsets -n apps ${MYSQL_STS_NAME} --replicas=0

During this time, the application should not be available.

➤ Execute the tool (after 10 - 15 seconds) again to verify that all prerequisites are fulfilled:

./eks-auto-mode-ebs-migration-tool \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --pvc-name ${ORIGINAL_PVC_NAME} \
  --namespace apps \
  --storageclass gp3-auto

➤ View the current PV and PVC state:

kubectl get pv,pvc -n apps

This should produce a result similar to the following:

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-c4bb4d38-3e92-4826-82d0-680c0d327bf7   10Gi       RWO            Retain           Bound    apps/data-retail-store-app-catalog-mysql-0   gp3            <unset>                          64m

NAME                                                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/data-retail-store-app-catalog-mysql-0   Bound    pvc-c4bb4d38-3e92-4826-82d0-680c0d327bf7   10Gi       RWO            gp3            <unset>                 64m
➤ Execute the tool in the "mutate" mode (--dry-run=false) and answer YES (note the capitalization) when prompted:

./eks-auto-mode-ebs-migration-tool \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --pvc-name ${ORIGINAL_PVC_NAME} \
  --namespace apps \
  --storageclass gp3-auto \
  --dry-run=false

➤ Once the migration is completed, view the current PV and PVC state:

kubectl get pv,pvc -n apps

The expected output should be similar to the previous one (both PV and PVC should have the Bound status), but with new IDs and a new, EKS Auto Mode CSI driver's storage class:

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                        STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/pvc-6ac68662-ac3b-43f0-9fd3-c73b59c2e899   10Gi       RWO            Retain           Bound    apps/data-retail-store-app-catalog-mysql-0   gp3-auto       <unset>                          23m

NAME                                                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/data-retail-store-app-catalog-mysql-0   Bound    pvc-6ac68662-ac3b-43f0-9fd3-c73b59c2e899   10Gi       RWO            gp3-auto       <unset>                 23m
Note that we have to execute the migration tool for each of the relevant PVCs, for example for data-retail-store-app-catalog-mysql-1, data-retail-store-app-catalog-mysql-2 etc...

After migrating all the PV and PVC objects to use the Auto Mode storage capability, we need to update our application (the catalog StatefulSet) to point to the new StorageClass that uses Auto Mode capability for storage too. Since the StatefulSet's storageClassName field is immutable, we will first need to delete the StatefulSet, and then reapply it with the right StorageClass configuration (and amount of replicas).

➤ Delete the StatefulSet of the catalog-mysql:

1
kubectl -n apps delete sts retail-store-app-catalog-mysql

We now will scale the application by updating the catalog component's Helm values.yml file, which would both migrate the MySQL Pod to use EKS Auto Mode compute and reset the number of replicas to 1.

For your applications, you should apply the above steps in a way that fits your configuration.

➤ Update the catalog-values.yml file for the catalog component (see here  for the whole values.yml file):

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
cat << EOF > catalog-values.yml
replicaCount: 3

nodeSelector:
  role: apps-auto-mode

tolerations:
  - key: role
    value: apps-auto-mode
    operator: Equal
    effect: NoSchedule

app:
  persistence:
    provider: mysql
    endpoint: ""
    database: "catalog"

    secret:
      create: true
      name: catalog-db
      username: catalog
      password: "mysqlcatalog123"

mysql:
  nodeSelector:
    role: apps-auto-mode
  tolerations:
    - key: role
      value: apps-auto-mode
      operator: Equal
      effect: NoSchedule
  persistentVolume:
    storageClass: "gp3-auto"
EOF

Pay attention to the new storageClass configuration of gp3-auto.

➤ Update the catalog component:

1
2
3
4
5
6
helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --namespace apps \
  --values catalog-values.yml \
  --reuse-values \
  --wait

➤ After a few moments, verify that we can access the application UI, via the ALB by executing the following command to extract the DNS name and Ctrl/Cmd-clicking on the URL in the output:

kubectl get ingress -n apps retail-store-app-ui-main-auto-mode \
  -o jsonpath="http://{.status.loadBalancer.ingress[*].hostname}{'\n'}"

➤ Verify that all pods are scheduled on the apps-auto-mode nodes by executing:

export APPS_KARPENTER_AUTO_MODE_NODES=$(kubectl get nodes -o json | jq -r '[.items[].metadata.labels | select(."karpenter.sh/nodepool" == "apps-auto-mode") | ."kubernetes.io/hostname"] | join("\\|")')
kubectl get pods -n apps -o wide | grep "${APPS_KARPENTER_AUTO_MODE_NODES}"

➤ View the distribution of instances across capacity types:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

The result should be similar to the following:

i-085c7fe5654a651c8                            |  us-west-2a  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0ce7c07eb76652c22                            |  us-west-2c  |  KARPENTER  |  nodepool:   apps-auto-mode
i-0d26a453b47d25d9f                            |  us-west-2b  |  KARPENTER  |  nodepool:   apps-auto-mode
...
ip-192-168-112-81.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  apps-mng
ip-192-168-150-83.us-west-2.compute.internal   |  us-west-2b  |  ON_DEMAND  |  apps-mng
ip-192-168-184-40.us-west-2.compute.internal   |  us-west-2c  |  ON_DEMAND  |  apps-mng
ip-192-168-133-200.us-west-2.compute.internal  |  us-west-2b  |  ON_DEMAND  |  system-mng  
ip-192-168-171-235.us-west-2.compute.internal  |  us-west-2c  |  ON_DEMAND  |  system-mng  
ip-192-168-99-105.us-west-2.compute.internal   |  us-west-2a  |  ON_DEMAND  |  system-mng  
We have now migrated our retail store application to EKS Auto Mode and all that remains is to remove the no-longer-required add-ons and compute resources that belong to the now-unused managed node groups!

Removing non-EKS Auto Mode Add-ons and Compute
Uninstall Karpenter | Uninstall the AWS Load Balancer Controller | Remove the Add-ons | Delete the Node Groups | Delete the Fargate Profile

After migrating to EKS Auto Mode, which includes several Kubernetes capabilities as core components we no longer require the following components and compute options:

The AWS Fargate apps profile
The apps-mng managed node group
The system-mng managed node group
The CoreDNS add-on
The Amazon VPC CNI add-on
The kube-proxy add-on
The EBS CSI Driver add-on
The EKS Pod Identity Agent add-on
The self-managed AWS Load Balancer Controller
The self-managed Karpenter
We will now remove these components from the migration cluster.

Uninstall the Self-Managed AWS Load Balancer Controller
➤ First, we'll uninstall the AWS Load Balancer Controller Helm chart and delete its CRDs:

helm uninstall --namespace kube-system aws-load-balancer-controller
kubectl delete -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

➤ Extract the AWS Load Balancer Controller's role Pod identity association:

export LBC_POD_IDENTITY_ASSOCIATION_ID=$(aws eks list-pod-identity-associations \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --query 'associations[*].associationId' \
  --output text)

➤ Now we'll detach and delete the AWS Load Balancer Controller policy:

export LBC_ROLE_NAME=$(aws eks describe-pod-identity-association \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --association-id ${LBC_POD_IDENTITY_ASSOCIATION_ID} | jq -r '.association.roleArn | split("/") | .[1]')

export LBC_POLICY_ARN=$(aws iam list-attached-role-policies \
  --role-name ${LBC_ROLE_NAME} | \
  jq -r '.AttachedPolicies[].PolicyArn')

aws iam detach-role-policy \
    --role-name ${LBC_ROLE_NAME} \
    --policy-arn ${LBC_POLICY_ARN}

aws iam delete-policy \
    --policy-arn ${LBC_POLICY_ARN}

➤ Finally, we'll delete the Pod identity association:

aws eks delete-pod-identity-association \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --association-id ${LBC_POD_IDENTITY_ASSOCIATION_ID}

Uninstall the Self-Managed Karpenter
➤ Verify that there are no self-managed Karpenter-created nodes:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

➤ Now let's delete the NodePool and NodeClass:

kubectl delete nodepool apps
kubectl delete ec2nodeclass apps

➤ Next, we'll uninstall Karpenter:

helm uninstall --namespace kube-system karpenter

➤ Delete the self-managed Karpenter nodes access entry:

aws eks delete-access-entry \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --principal-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${MIGRATION_CLUSTER_NAME}

➤ Extract the Karpenter role Pod identity association:

export KARPENTER_POD_IDENTITY_ASSOCIATION_ID=$(aws eks list-pod-identity-associations \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --namespace kube-system \
  --service-account karpenter \
  --query 'associations[*].associationId' \
  --output text)

➤ Extract the Karpenter controller role name:

export KARPENTER_ROLE_NAME=$(aws eks describe-pod-identity-association \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --association-id ${KARPENTER_POD_IDENTITY_ASSOCIATION_ID} | jq -r '.association.roleArn | split("/") | .[1]')

➤ Detach the Karpenter policy:

aws iam detach-role-policy \
  --role-name ${KARPENTER_ROLE_NAME} \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${MIGRATION_CLUSTER_NAME}

➤ Delete the Pod identity association:

aws eks delete-pod-identity-association \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --association-id ${KARPENTER_POD_IDENTITY_ASSOCIATION_ID}

➤ Remove Karpenter-related instance profiles:

KARPENTER_NODE_ROLE=KarpenterNodeRole-${MIGRATION_CLUSTER_NAME}
KARPENTER_INSTANCE_PROFILES=$(aws iam list-instance-profiles-for-role \
  --role-name ${KARPENTER_NODE_ROLE} \
  --query 'InstanceProfiles[*].InstanceProfileName' \
  --output text)

for profile in ${KARPENTER_INSTANCE_PROFILES}; do
  echo "Removing role from instance profile ${profile}"
  aws iam remove-role-from-instance-profile --instance-profile-name "${profile}" --role-name ${KARPENTER_NODE_ROLE}
  echo "Deleting instance profile ${profile}"
  aws iam delete-instance-profile --instance-profile-name "${profile}"
done

➤ Delete the Karpenter role:

aws iam delete-role --role-name ${KARPENTER_ROLE_NAME}

➤ Remove the remaining Karpenter-related resources:

aws cloudformation delete-stack --stack-name Karpenter-${MIGRATION_CLUSTER_NAME}
aws cloudformation wait stack-delete-complete --stack-name Karpenter-${MIGRATION_CLUSTER_NAME}

aws ec2 describe-launch-templates --filters "Name=tag:karpenter.k8s.aws/cluster,Values=${MIGRATION_CLUSTER_NAME}" |
    jq -r ".LaunchTemplates[].LaunchTemplateName" |
    xargs -I{} aws ec2 delete-launch-template --launch-template-name {}

Remove the Unnecessary Amazon EKS Add-ons
➤ Now we'll remove the Amazon EKS Add-ons that are no longer required since they are provided as core components in EKS Auto Mode:

aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name metrics-server
aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name coredns
aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name vpc-cni
aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name kube-proxy
aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name aws-ebs-csi-driver
aws eks delete-addon --cluster-name ${MIGRATION_CLUSTER_NAME} --addon-name eks-pod-identity-agent

Delete the Managed Node Groups
We have now successfully migrated the application and removed all the unnecessary controllers and add-ons. With EKS Auto Mode handling our compute needs, we no longer require either of the managed node groups.

➤ Let's delete the node groups:

eksctl delete nodegroup \
  --region ${AWS_REGION} \
  --cluster ${MIGRATION_CLUSTER_NAME} \
  --name=apps-mng

eksctl delete nodegroup \
  --region ${AWS_REGION} \
  --cluster ${MIGRATION_CLUSTER_NAME} \
  --name=system-mng

Removal of the node groups and termination of their instances may take a couple of minutes.

➤ Verify that all non-EKS Auto Mode compute options were removed:

kubectl get nodes -o json | jq -r '.items[].metadata.labels | ."kubernetes.io/hostname" + " | " + ."topology.kubernetes.io/zone" + " | " + (."eks.amazonaws.com/capacityType" // if ."eks.amazonaws.com/compute-type" == "fargate" then "FARGATE" else "KARPENTER" end) + " | " + (."eks.amazonaws.com/nodegroup" // if ."karpenter.sh/nodepool" then "nodepool: " + ."karpenter.sh/nodepool" else "profile: apps" end)' | column -t | sort -k 5

Note that we've created the node groups using eksctl, so your process may slightly differ.

Delete the Fargate Profile
➤ Finally, let's delete the Fargate profile that is no longer needed:

aws eks delete-fargate-profile \
  --cluster-name ${MIGRATION_CLUSTER_NAME} \
  --fargate-profile-name apps
