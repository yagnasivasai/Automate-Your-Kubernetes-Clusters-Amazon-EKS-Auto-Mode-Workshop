
https://catalog.us-east-1.prod.workshops.aws/join?access-code=e50e-02db7f-b3
https://docs.aws.amazon.com/eks/latest/userguide/set-builtin-node-pools.html
https://github.com/doitintl/kube-no-trouble

kubectx arn:aws:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/demo-cluster
kubectl get crds


for POLICY in \
  "arn:aws:iam::aws:policy/AmazonEKSComputePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSBlockStoragePolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSLoadBalancingPolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSNetworkingPolicy" \
  "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
do
  echo "Attaching policy ${POLICY} to IAM role ${DEMO_CLUSTER_ROLE_NAME}..."
  aws iam attach-role-policy --role-name ${DEMO_CLUSTER_ROLE_NAME} --policy-arn ${POLICY}
done

aws iam update-assume-role-policy --role-name $DEMO_CLUSTER_ROLE_NAME --policy-document '{
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
}'



aws iam get-role --role-name ${DEMO_CLUSTER_ROLE_NAME} | \
  jq -r '.Role.AssumeRolePolicyDocument.Statement[].Action[]'

aws iam list-attached-role-policies --role-name ${DEMO_CLUSTER_ROLE_NAME} | \
  jq -r '.AttachedPolicies[].PolicyName'


aws eks update-cluster-config \
    --name ${DEMO_CLUSTER_NAME} \
    --compute-config enabled=true,nodeRoleArn=${DEMO_CLUSTER_NODE_ROLE_ARN},nodePools=system,general-purpose \
    --kubernetes-network-config '{"elasticLoadBalancing":{"enabled": true}}' \
    --storage-config '{"blockStorage":{"enabled": true}}'


sleep 10; aws eks describe-cluster --name ${DEMO_CLUSTER_NAME} --query 'cluster.status'

aws eks wait cluster-active --name ${DEMO_CLUSTER_NAME}

aws eks describe-cluster --name ${DEMO_CLUSTER_NAME} --query 'cluster.status'


cat << EOF > ~/environment/values-ui.yaml

app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80
EOF

helm upgrade -i retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-orders oci://public.ecr.aws/aws-containers/retail-store-sample-orders-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-carts oci://public.ecr.aws/aws-containers/retail-store-sample-cart-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-checkout oci://public.ecr.aws/aws-containers/retail-store-sample-checkout-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes
helm upgrade -i retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} -f ~/environment/values-ui.yaml --hide-notes


kubectl wait --for=condition=Ready nodes --all
kubectl get pods,svc


kubectl get nodes -l karpenter.sh/nodepool=general-purpose
kubectl get nodes -l karpenter.sh/nodepool=system


for node in $(kubectl get nodes -l karpenter.sh/nodepool=general-purpose -o custom-columns=NAME:.metadata.name --no-headers); do
  echo "Pods on $node:"
  kubectl get pods --all-namespaces --field-selector spec.nodeName=$node
done

kubectl scale --replicas=12 deployment/retail-store-app-ui

for node in $(kubectl get nodes -l karpenter.sh/nodepool=general-purpose -o custom-columns=NAME:.metadata.name --no-headers); do
  echo "Pods on $node:"
  kubectl get pods --all-namespaces --field-selector spec.nodeName=$node
done



cat  << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 3
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ui
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

kubectl scale --replicas=12 deployment/retail-store-app-ui
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=retail-store-app-ui --namespace default --timeout=300s

kubectl get node -L topology.kubernetes.io/zone --no-headers | while read node status roles age version zone; do
echo "Pods on node $node (Zone: $zone):"
  kubectl get pods --all-namespaces --field-selector spec.nodeName=$node -l app.kubernetes.io/instance=retail-store-app-ui
echo "-----------------------------------"
done

kubectl get nodepools general-purpose -o yaml

eksctl create addon --name metrics-server --cluster ${DEMO_CLUSTER_NAME}
kubectl get deployment metrics-server -n kube-system

kubectl top node
kubectl top pods -l app.kubernetes.io/name=ui

kubectl get hpa --all-namespaces

cat << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 3
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ui
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/instance: retail-store-app-ui

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 80
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

kubectl get hpa  

kubectl run load-generator \
 --image=williamyeh/hey:latest \
 --restart=Never -- -c 10 -q 10 -z 4m http://retail-store-app-ui/utility/stress/200000

kubectl get hpa retail-store-app-ui --watch
watch -t kubectl get pods -l app.kubernetes.io/instance=retail-store-app-ui
watch -t kubectl get nodes

kubectl get nodepool general-purpose -o yaml
kubectl get nodes -L kubernetes.io/arch

cat << EOF >~/environment/nodepool-graviton.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: graviton
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    budgets:
    - nodes: 10%
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata: {}
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      - key: eks.amazonaws.com/instance-category
        operator: In
        values:
        - c
        - m
        - r
      - key: eks.amazonaws.com/instance-generation
        operator: Gt
        values:
        - "4"
      - key: kubernetes.io/arch
        operator: In
        values:
        - arm64
      taints:
      - effect: NoSchedule
        key: GravitonOnly
      terminationGracePeriod: 24h0m0s
  limits:
    cpu: "1000"
    memory: 1000Gi
EOF

kubectl apply -f ~/environment/nodepool-graviton.yaml

kubectl describe pod --selector app.kubernetes.io/name=ui



cat << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: ui

autoscaling:
  enabled: false
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector:
  karpenter.sh/nodepool: graviton
tolerations:
- key: "GravitonOnly"
  operator: "Exists"
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=retail-store-app-ui --namespace default --timeout=300s
kubectl get pods -l app.kubernetes.io/instance=retail-store-app-ui

kubectl get nodes -L kubernetes.io/arch -L karpenter.sh/nodepool
kubectl get pods -l app.kubernetes.io/name=ui -o wide

cat << EOF >~/environment/nodepool-ondemandspotsplit.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: ondemand
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata:
      labels:
        EKSAutoNodePool: OnDemandSpotSplit
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: eks.amazonaws.com/instance-category
        operator: In
        values:
        - c
        - m
        - r
      - key: eks.amazonaws.com/instance-generation
        operator: Gt
        values:
        - "4"
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      - key: capacity-spread
        operator: In
        values:
        - "1"
      taints:
      - effect: NoSchedule
        key: OnDemandSpotSplit
      terminationGracePeriod: 24h0m0s
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot
  labels:
    app.kubernetes.io/managed-by: app-team
spec:
  disruption:
    consolidateAfter: 30s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    metadata:
      labels:
        EKSAutoNodePool: OnDemandSpotSplit
    spec:
      expireAfter: 336h
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: eks.amazonaws.com/instance-category
        operator: In
        values:
        - c
        - m
        - r
      - key: eks.amazonaws.com/instance-generation
        operator: Gt
        values:
        - "4"
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - spot
      - key: capacity-spread
        operator: In
        values:
        - "2"
        - "3"
        - "4"
        - "5"
      taints:
      - effect: NoSchedule
        key: OnDemandSpotSplit
      terminationGracePeriod: 24h0m0s
EOF

kubectl apply -f ~/environment/nodepool-ondemandspotsplit.yaml

cat << EOF >~/environment/values-catalog.yaml
replicaCount: 5
  
topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 5
    topologyKey: capacity-spread
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: catalog

nodeSelector:
  EKSAutoNodePool: OnDemandSpotSplit
tolerations:
- key: "OnDemandSpotSplit"
  operator: "Exists"
EOF

helm upgrade -f ~/environment/values-catalog.yaml retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

cat << EOF >~/environment/values-catalog.yaml
replicaCount: 5
  
topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 5
    topologyKey: capacity-spread
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: catalog

nodeSelector:
  EKSAutoNodePool: OnDemandSpotSplit
tolerations:
- key: "OnDemandSpotSplit"
  operator: "Exists"
EOF

helm upgrade -f ~/environment/values-catalog.yaml retail-store-app-catalog oci://public.ecr.aws/aws-containers/retail-store-sample-catalog-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

kubectl get node -L karpenter.sh/capacity-type --no-headers | while read node status roles age version capacity_type; do
echo "Pods on node $node (Capacity Type: $capacity_type):"
  kubectl get pods --all-namespaces --field-selector spec.nodeName=$node -l app.kubernetes.io/instance=retail-store-app-catalog
echo "-----------------------------------"
done

Expose the application
EKS Auto Mode simplifies the process by automatically managing the lifecycle of the ALBs and NLBs that are required for our application. As EKS Auto Mode is Kubernetes conformant, it allows us to use the same Kubernetes constructs of Service  and Ingress  to provision those Load Balancers.

In the remainder of this module, we'll learn how to provision those load balancers with Auto Mode.

Step 1: Set Up IngressClass for ALB
Since Ingresses can be implemented by different controllers, each Ingress should specify a class, a reference to an IngressClass resource that contains additional configuration including the name of the controller that should implement the class.

IngressClass resources contain an optional parameters field. This can be used to reference additional implementation-specific configuration for this class.

To target the EKS Auto Mode ALB load balancing capability controller, we will create an IngressClassParams, which allows us to define an AWS specific configuration for our ALB such as certificates to use, the subnets to use for the ALB ENIs, or the ingress group  configuration to group together multiple ingress objects into a single ALB.

The supported configurations for the IngressClassParams objects are listed  in EKS Auto Mode documentation. Additionally, we will create IngressClass that will use the IngressClassParams and point to the EKS Auto Mode capability. This is a one-time setup required for using ALBs in our cluster. Notice the spec.controller definition in the IngressClass below.

➤ Create the IngressClass and IngressClassParams to further configure the EKS Auto Mode load balancing capability:

cat << EOF >~/environment/ingress.yaml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-alb
spec:
  scheme: internet-facing
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-alb
EOF

kubectl apply -f ~/environment/ingress.yaml

kubectl get ingressclass,ingressclassparams

Step 2: Deploy the Retail Store Application
We will now update the UI component to provision an ALB by creating an Ingress object, as we can see with the ingress configuration in the component's Helm chart values.

➤ Re-deploy the UI component with custom values:

cat << EOF >~/environment/values-ui.yaml
app:
  theme: default
  endpoints:
    catalog: http://retail-store-app-catalog:80
    carts: http://retail-store-app-carts:80
    checkout: http://retail-store-app-checkout:80
    orders: http://retail-store-app-orders:80

replicaCount: 3

autoscaling:
  enabled: false
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

topologySpreadConstraints:
   - maxSkew: 1
     minDomains: 3
     topologyKey: topology.kubernetes.io/zone
     whenUnsatisfiable: DoNotSchedule
     labelSelector:
       matchLabels:
         app.kubernetes.io/name: ui

ingress:
  enabled: true
  className: eks-auto-alb
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    alb.ingress.kubernetes.io/success-codes: '200-399'
EOF

helm upgrade -f ~/environment/values-ui.yaml retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} --hide-notes

kubectl wait --for=condition=available deployments retail-store-app-ui --all

Step 3: Access the UI application with the provisioned ALB
The Ingress object that we've deployed in the previous step gets translated by EKS Auto Mode into an ALB with the appropriate configurations we've defined in the Ingress object itself, and in the IngressClassParams above.

export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$ALB_URL"'`].LoadBalancerArn' --output text)

echo "Your application is available at: http://${ALB_URL}"


Step 4: Expose Catalog Service Using NLB
In the previous steps, we've used EKS Auto Mode to provision an ALB. We'll now experience how to use EKS Auto Mode to provision NLB using the Kubernetes Service object.

➤ Execute the following command.

kubectl apply -f - << EOF
apiVersion: v1
kind: Service
metadata:
  name: catalog-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
  selector:
    app.kubernetes.io/name: catalog
EOF

 Wait for the NLB to be provisioned and get its URL:

export NLB_URL=$(kubectl get service catalog-nlb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?
DNSName==`'"$NLB_URL"'`].LoadBalancerArn' --output text)
echo "The catalog service is also available at: http://${NLB_URL}"

Step 5.1: Test Application Load Balancer Access
Let's test access to our application and observe load balancing in action. The catalog service has been configured with 5 replicas in the compute module under "On-Demand & Spot Split Ratio". We will use this to demonstrate load distribution.

➤ First, verify that all catalog pods are running:
kubectl get pods -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service
kubectl logs -f -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service --prefix=true

 In another terminal, generate some traffic through the ALB:
# Get the ALB URL
export ALB_URL=$(kubectl get ingress retail-store-app-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "The application is available at: http://${ALB_URL}"

# Generate traffic to see load balancing across pods
CURL_CMD=$(which curl)
for i in {1..15}; do
  echo "Sending request $i..."
  $CURL_CMD -s "http://${ALB_URL}/catalog?request=$i" > /dev/null
  sleep 2
done

You should see detailed logs in your first terminal showing requests being distributed across all three catalog pods. Each log line shows:

Which Pod handled the request (in the prefix)
HTTP method and path
Response status code
Request processing time
Client IP address
Example log output showing distribution across pods:

Step 5.2: Test Network Load Balancer Access
➤ To test the access to the catalog service directly through the NLB provisioned earlier, ensure that the kubectl logs command is still running on the other terminal

kubectl logs -f -l app.kubernetes.io/name=catalog,app.kubernetes.io/component=service --prefix=true

Now we can test from the access to the catalog service through the NLB by using the NLB DNS name with the appended URI below:
export NLB_URL=$(kubectl get service catalog-nlb -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://${NLB_URL}/catalog/products | jq

You should expect to see a JSON response from the catalog service, as well as logs from that request on the other terminal.

Step 6: Share ALB Across Multiple Services - Multiple Ingress Pattern
In some use-cases we need to ensure that multiple ingress objects don't create multiple ALBs but rather use the same ALB with multiple routing rules. This is where EKS Auto Mode supports ingress grouping using the IngressClassParams object (see reference in the documentation ). In this step, we will create a new IngressClass object with new IngressClassParams that supports such grouping. For demonstration purposes, we will then create 2 ingress objects: one for the ui component and one for the catalog service. With this configuration, since we've configured the grouping on the IngressClassParams, a single ALB will be created pointing to both of those services. Follow the steps below to achieve that:

➤ 1. Create a new IngressClassParams and IngressClass with group.name configuration:

cat << EOF >~/environment/ingress-class-group.yaml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: eks-auto-alb-group-retail
spec:
  scheme: internet-facing
  group:
    name: retail
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: eks-auto-alb-group-retail
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: eks-auto-alb-group-retail
EOF

kubectl apply -f ~/environment/ingress-class-group.yaml

➤ 2. Create an ingress object for the ui component (note the use of the newly created IngressClass eks-auto-alb-group-retail ):
kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-shared-group-ui
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
spec:
  ingressClassName: eks-auto-alb-group-retail
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: retail-store-app-ui
                port:
                  number: 80
EOF

kubectl apply -f - << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: retail-store-shared-group-catalog
  annotations:
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ingressClassName: eks-auto-alb-group-retail
  rules:
  - http:
      paths:
      - path: /catalog
        pathType: Prefix
        backend:
          service:
            name: retail-store-app-catalog
            port:
              number: 80
EOF

kubectl get ingress

 5. Extract the ALB DNS (note that because both ingresses have the same DNS, we can randomly choose one of them):
# Get the shared ALB URL
export SHARED_ALB_URL=$(kubectl get ingress retail-store-shared-group-ui -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
# wait for the shared ALB to become active
aws elbv2 wait load-balancer-available --load-balancer-arns $(aws elbv2 describe-load-balancers --query 'LoadBalancers[?DNSName==`'"$SHARED_ALB_URL"'`].LoadBalancerArn' --output text)
echo "The shared ALB is available at: http://$SHARED_ALB_URL"

6. Access the application via the shared ALB URL, where you should expect an HTML output from the / path (coming from the ui component):
curl -s "http://$SHARED_ALB_URL/"

7. Access the the /catalog/products path, where you should expect a JSON response with a list of catalog items:
curl -s "http://$SHARED_ALB_URL/catalog/products" | jq


Amazon EKS Auto Mode simplifies and automates critical networking tasks for pod and service connectivity by managing the VPC Container Network Interface (CNI) configuration and load balancer provisioning for the cluster.

In modern environments there are several additional use cases that require advanced network configuration.

IP exhaustion
By default, Amazon VPC CNI will assign pods an IP address selected from the primary subnet. The primary subnet is the subnet CIDR that the primary ENI is attached to, usually the subnet of the node/host.

If the subnet CIDR is too small, the CNI may not be able to acquire enough secondary IP addresses to assign to the pods, which is a common challenge for EKS IPv4 clusters.

Additionally, some workloads require to increase pod density  in order to improve resources utilization. In Amazon EKS this is implemented by enabling VPC CNI prefix mode . To implement the prefix mode, instead of a single IP address, VPC CNI configures EC2 to assign /28 IP prefixes (16 IP addresses) to the ENI IP slots. When EC2 allocates a /28 IPv4 prefix to an ENI, it has to be a contiguous block of IP addresses from your subnet. If the subnet is fragmented due to an increased usage of the subnet by AWS services and worker nodes themselves, the prefix attachment may fail, essentially reducing the IP address space utilization.

Infrastructure and application traffic separation
Creating distinct network paths for different types of communication within a Kubernetes cluster is particularly valuable for organizations that need to maintain clear boundaries between their infrastructure management communications and their application workloads. Node-to-node communication typically includes cluster management traffic, infrastructure monitoring, while pod-to-pod communication handles application-specific data flows and service interactions.

Different network configuration for node and pod subnets
Applying a different network configuration to nodes and pods is also a common requirement for more complex systems. This includes customizing network address translation (SNAT) policies, placement on worker nodes and pods in public or private subnets, and defining different tagging to satisfy tools requirements.

Security considerations
The last two use cases are especially relevant use cases where traffic and access control are crucial to the security of the system.

Addressing the use cases
EKS Auto Mode provides advanced networking capabilities  that allow us to implement granular network controls and traffic separation as well as multiple layers of network security utilizing the standard VPC features  and Kubernetes-native network policies .

In this lab, we will show how to simplify solutions that address IP exhaustion and pod network isolation using EKS Auto Mode advanced networking capabilities. We will do so by:

Adding a secondary CIDR block to the cluster VPC
Creating new subnets from the new CIDR block
Targeting the new subnets via EKS AutoMode NodePool and NodeClass configuration
Configuring the application components to utilize the above configuration

Handle IP exhaustion
Review the EKS Auto Mode configuration
➤ Review the current nodes and pods IPs:

kubectl get nodes -o custom-columns=NAME:.metadata.name,INTERNAL-IP:.status.addresses[0].address
kubectl get pods -o wide

Add a secondary CIDR to the cluster VPC
➤ Execute the following command to store the VPC ID in the terminal:

export VPC_ID=$(aws eks describe-cluster --name $DEMO_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)

➤ In the same terminal, execute the following command to explore all the CIDRs attached to the cluster VPC:

aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

Add a secondary CIDR to the cluster VPC
➤ Execute the following command to store the VPC ID in the terminal:

export VPC_ID=$(aws eks describe-cluster --name $DEMO_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)

➤ In the same terminal, execute the following command to explore all the CIDRs attached to the cluster VPC:

aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

Once again, as expected, there is only one, original, CIDR.

A reminder: EKS Auto Mode enables  VPC CNI prefix delegation by default.

For the sake of the workshop, let's assume that we've exhausted enough IPs from the original CIDR block, so that it's impossible to provision new /28 blocks, which we require to reduce latency of pod provision or to increase pod density on our worker nodes.

The most straightforward way of dealing with the IP exhaustion issue is to attach a secondary CIDR to our VPC, create and tag subnets from that secondary CIDR and configure EKS to provision nodes and pods from these subnets.

Let's attach a secondary CIDR to the cluster VPC. Our options are outlined in this document . Since we've already used the entire 192.168.0.0/16 block, we need to select a different one.

➤ In the same terminal as the commands above (as we require the VPC_ID) execute the following command:

aws ec2 associate-vpc-cidr-block \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.0.0.0/16

The above command is expected to fail with the following error:

An error occurred (InvalidVpc.Range) when calling the AssociateVpcCidrBlock operation: The CIDR '10.0.0.0/16' is restricted. Use a CIDR from the same private address range as the current VPC CIDR, or use a publicly-routable CIDR.
For additional restrictions, see https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html
This is because, as outlined in the VPC CIDR selection restrictions document , you can not combine different CIDRs from different RFC 1918  blocks.

To resolve this and create more private IP address space for our pods we can use  the 100.64.0.0/10 block instead.

➤ Execute the following command:

aws ec2 associate-vpc-cidr-block \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.0.0/16

This should succeed now and produce the following output:

{
    "CidrBlockAssociation": {
        "AssociationId": "vpc-cidr-assoc-0df9d27b82606badc",
        "CidrBlock": "100.64.0.0/16",
        "CidrBlockState": {
            "State": "associating"
        }
    },
    "VpcId": "vpc-0413d277b700882f4"
}
➤ After a moment we can verify that the CIDR has been successfully attached to the cluster vpc by executing:

aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[].CidrBlock'

The output should now contain both the original and the new CIDR:

[
    "192.168.0.0/16",
    "100.64.0.0/16"
]
Create additional subnets in the cluster VPC
We can now create 3 subnets in 3 different Availability Zones from the new secondary CIDR block, as recommended by the resilience best practices.

➤ Execute the following:

export VPC_ID=$(aws eks describe-cluster --name $DEMO_CLUSTER_NAME --query 'cluster.resourcesVpcConfig.vpcId' --output text)

export SUBNET_ID_A=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.0.0/19 \
  --availability-zone ${AWS_REGION}a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2a},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

export SUBNET_ID_B=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.32.0/19 \
  --availability-zone ${AWS_REGION}b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2b},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

export SUBNET_ID_C=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 100.64.64.0/19 \
  --availability-zone ${AWS_REGION}c \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=eks-subnet-2c},{Key=advanced-networking,Value=1}]' \
  --query 'Subnet.SubnetId' \
  --output text)

➤ Verify that the subnets have been created properly:

aws ec2 describe-subnets \
  --filters "Name=tag:advanced-networking,Values=1" \
  --query "Subnets[*].CidrBlock"

This should produce the following output:

[
    "100.64.64.0/19",
    "100.64.32.0/19",
    "100.64.0.0/19"
]
For pods to communicate with external resources, we need to associate our new subnets with a route table that defines the required configuration. In this case we can simply use the same route table we've used for the rest of the subnets.

➤ Execute the following (in that same terminal):

export ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=${VPC_ID}" "Name=route.nat-gateway-id,Values=nat-*" \
  --query 'RouteTables[0].RouteTableId' \
  --output text)

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_A}

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_B}

aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID_C}

Note that assigning a new VPC CIDR automatically updated the main route table to designate the 100.64.0.0/16 as a local target, in the same manner it does for the original 192.168.0.0/16 CIDR.

This expected output is as follows:

{
    "AssociationId": "rtbassoc-0bf6e8322df34a5a9",
    "AssociationState": {
        "State": "associated"
    }
}
{
    "AssociationId": "rtbassoc-0c742c5288fc4a532",
    "AssociationState": {
        "State": "associated"
    }
}
{
    "AssociationId": "rtbassoc-069e88b0bbe1d0fc0",
    "AssociationState": {
        "State": "associated"
    }
}
If we were to deploy an application, for its pods to be scheduled on instances provisioned in the new subnets, it would not actually work.

This is because Auto Mode autoscaling mechanism, via the built-in default NodeClass, doesn't "know" about them.

Create a custom NodeClass and NodePool to utilize the new subnets
To introduce the subnets and to make sure that network communication and connection to AWS services would work properly, we will target the corresponding tag (advanced-networking: '1', highlighted below), while re-using the IAM role and the original security group that allows the traffic between pods and the control plane.

➤ Create a new NodeClass that targets the new subnets:

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
cat << EOF > ~/environment/advanced-networking-nodeclass.yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: advanced-networking
spec:
  role: '${DEMO_CLUSTER_NODE_ROLE_NAME}'
  subnetSelectorTerms:
    - tags:
        advanced-networking: '1'
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
EOF

➤ Create a new NodePool that uses the NodeClass above:

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
cat << EOF > ~/environment/advanced-networking-nodepool.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: advanced-networking
spec:
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 5m
  template:
    metadata:
      labels:
        role: advanced-networking
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: advanced-networking
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand]
        - key: eks.amazonaws.com/instance-category
          operator: In
          values: [c, m, r]
        - key: eks.amazonaws.com/instance-cpu
          operator: In
          values: ['4', '8', '16', '32']
EOF

Note that we've added a custom label to the NodePool above to allow targeting these specific worker nodes with a nodeSelector in one of our application components. We only do the explicit targeting to demonstrate that these subnets are fully operational. In a real-world scenario new nodes and pods would consume IPs from the new subnets as required – most notably when there are no more IPs in the original subnets.

➤ Deploy the NodePool and the NodeClass:

kubectl apply -f ~/environment/advanced-networking-nodeclass.yaml
kubectl apply -f ~/environment/advanced-networking-nodepool.yaml

➤ Verify that the created components are ready to be used (the value in the READY column is True, which may take a couple of seconds):

kubectl get nodepool,nodeclass

To illustrate pods being provisioned in the new subnets, we'll re-deploy the UI application component.

➤ Execute the following command to create a custom values.yaml file:

1
2
3
4
cat << EOF > ~/environment/advanced-networking-values-ui.yaml
nodeSelector:
  role: advanced-networking
EOF

We added the node selector to illustrate topology spread across the new subnets to the file above and we will re-use the rest of the values from the previous Helm chart installation.

➤ Re-deploy the UI component:

1
2
3
4
5
helm upgrade retail-store-app-ui oci://public.ecr.aws/aws-containers/retail-store-sample-ui-chart \
  --version ${RETAIL_STORE_APP_HELM_CHART_VERSION} \
  --values ~/environment/advanced-networking-values-ui.yaml \
  --reuse-values \
  --wait

Note that it will take a minute or so for the new instances to become operational.

➤ We can verify that both the UI pods and their nodes received IPs from the new subnets:

kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address -l role=advanced-networking
kubectl get pods -l app.kubernetes.io/name=ui -o wide

This should provide an output similar to the following:

NAME                  IP
i-02b5a3d625f34288b   100.64.26.164
i-042c5454735eab393   100.64.58.201
i-0fcfb02997484b44c   100.64.84.98
NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE
retail-store-app-ui-69db5c4cdc-2tr6s   1/1     Running   0          20m   100.64.47.144   i-042c5454735eab393
retail-store-app-ui-69db5c4cdc-676hh   1/1     Running   0          25m   100.64.3.16     i-02b5a3d625f34288b
retail-store-app-ui-69db5c4cdc-wr8ll   1/1     Running   0          20m   100.64.79.48    i-0fcfb02997484b44c
We have now configured our subnets and the corresponding Auto Mode components to address the common IP exhaustion use case.

Sometimes, there may be additional considerations, such as traffic separation or application of different security controls between nodes and pods, that require us to take the network configuration and create a pod network isolation.

Isolate Pod Network
EKS Auto Mode allows to address these requirements by using subnet selection for pods  NodeClass configuration.

➤ Update the advanced-networking NodeClass by executing:

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
cat << EOF > ~/environment/advanced-networking-nodeclass.yaml
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: advanced-networking
spec:
  role: '${DEMO_CLUSTER_NODE_ROLE_NAME}'
  subnetSelectorTerms:
    - tags:
        kubernetes.io/role/internal-elb: '1'
  securityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
  podSubnetSelectorTerms:
    - tags:
        advanced-networking: '1'
  podSecurityGroupSelectorTerms:
    - tags:
        kubernetes.io/cluster/demo-cluster: owned
EOF

The code above (see the highlighted lines) ensures that nodes and pods are placed into different subnets, while still allowing control plane and node-to-pod communications. We achieved that by configuring:

the node-level subnet selector to target the original subnets
the pod-level subnet selector (podSubnetSelectorTerms) to target the new subnets we created earlier
the security group selector for both nodes and pods (identical in this example) to target a shared security group that allows traffic between the control plane, nodes and (now) pods
Note that for pods we've specifically targeted the private subnets, using kubernetes.io/role/elb: 1 tag as outlined in the documentation .

Note that using podSecurityGroupSelectorTerms is mandatory when using podSubnetSelectorTerms configuration. Alternatively, we could have used a different security group to also address the traffic separation use case in the same configuration.

Also note that EKS Auto Mode doesn't support  Security Groups per Pod (SGPP).

Finally, keep in mind the following considerations for subnet selectors for pods:

Reduced pod density: fewer pods can run on each node, because the IP slots on the node's primary EMI can no longer be used for pods
Routing configuration: route table and network Access Control List (ACL) of the pod subnets are properly configured to allow the required communications
➤ Deploy the NodeClass (the advanced-networking NodePool doesn't require any changes):

kubectl apply -f ~/environment/advanced-networking-nodeclass.yaml

➤ Verify that the created components are ready to be used (the value in the READY column is True, which may take a couple of seconds):

kubectl get nodepool,nodeclass

Once applied, we don't actually need to do anything else, as Auto Mode will detect the drift  (difference between the cluster state and the configuration outlined in the NodeClass) and reconcile the cluster to the desired state – node and pod subnet separation as we defined.

➤ Verify that the UI pods and their nodes received IPs from different subnets:

kubectl get nodes -o custom-columns=NAME:.metadata.name,IP:.status.addresses[0].address -l role=advanced-networking
kubectl get pods -l app.kubernetes.io/name=ui -o wide

Note that it will take a couple of minutes for the new non-drifted images to become operational.

This should provide an output similar to the following, showing nodes and pods IPs indeed belong to different subnets:

NAME                  IP
i-03fed7a34057e58c2   192.168.164.36
i-042f5d093908ac461   192.168.105.230
i-0a126a6040e63ec87   192.168.154.221
NAME                                   READY   STATUS    RESTARTS   AGE     IP              NODE
retail-store-app-ui-69db5c4cdc-knhv8   1/1     Running   0          5m22s   100.64.34.96    i-0a126a6040e63ec87
retail-store-app-ui-69db5c4cdc-nvtnh   1/1     Running   0          7m11s   100.64.4.96     i-042f5d093908ac461
retail-store-app-ui-69db5c4cdc-vgqm5   1/1     Running   0          6m21s   100.64.81.145   i-03fed7a34057e58c2
Summary
In this lab, we've learned how to address common IP exhaustion and network separation use cases by performing the following:

extending the cluster VPC with secondary CIDR blocks to provide additional IP address space
creating new subnets in the secondary CIDR block
associating a route table to ensure subnet-to-subnet traffic
configuring EKS AutoMode NodePool and NodeClass resources to implement advanced networking (with or without podSubnetSelectorTerms and podSecurityGroupSelectorTerms)
