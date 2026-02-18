We'll be using an AWS Region where Amazon EKS is available. The workshop will use the default region configured on your machine/laptop.

To get started, we'll need:

Access to an AWS account
Permission to deploy CloudFormation stacks that create and manage resources including: IAM, VPC, Lambda, CodeBuild, EKS, EC2, and CloudFront
Service Linked Roles enabled for ElasticLoadBalancing and EC2Spot services
Let's enable the required Service Linked Roles:

aws iam create-service-linked-role --aws-service-name elasticloadbalancing.amazonaws.com  
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
