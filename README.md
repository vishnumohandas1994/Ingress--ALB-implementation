# Ingress--ALB-implementation
AWS Load Balancer Controller on EKS (with Pod Identity)
Step-01 

1.Create a trust policy file for the Load Balancer Controller IAM Role.

2.Create and attach the AWSLoadBalancerControllerIAMPolicy to that role.

3.Create an EKS Pod Identity Association between the IAM Role and ServiceAccount.

4.Install the AWS Load Balancer Controller using Helm.

5.Verify successful deployment.

Note: EKS cluster created is with private node group 

Step-02-02: Create IAM Policy for LBC
Download officeial IAM policy 
curl -o aws-load-balancer-controller-policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy_{EKS_CLUSTER_NAME} \
  --policy-document file://aws-load-balancer-controller-policy.json

  Step-02-03: Create Trust Policy File

  To create the trus policy used below policy 
  {
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}

Attach above policy to the IAM role 

aws iam create-role \
  --role-name AmazonEKS_LBC_Role_${EKS_CLUSTER_NAME} \
  --assume-role-policy-document file://aws-load-balancer-controller-trust-policy.json

# Attach the LBC IAM Policy
aws iam attach-role-policy \
  --role-name AmazonEKS_LBC_Role_${EKS_CLUSTER_NAME} \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy_${EKS_CLUSTER_NAME}
Step-02-05: Create EKS Pod Identity Association

aws eks create-pod-identity-association \
  --cluster-name ${EKS_CLUSTER_NAME} \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_LBC_Role_${EKS_CLUSTER_NAME}

Step-03 – Install AWS Load Balancer Controller (Helm)
Step-03-01: Add Helm Repo and Update
helm repo add eks https://aws.github.io/eks-charts
helm repo update
Step-03-02: Install Load Balancer Controller
# Get VPC ID
VPC_ID=$(aws eks describe-cluster \
  --name ${EKS_CLUSTER_NAME} \
  --query "cluster.resourcesVpcConfig.vpcId" \
  --output text)

# Verify VPC ID
echo $VPC_ID

# Install AWS Load Balancer Controller using HELM
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${EKS_CLUSTER_NAME} \
  --set region=${AWS_REGION} \
  --set vpcId=${VPC_ID} \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller  
✅ Explanation:

serviceAccount.create=true → Creates the ServiceAccount automatically during Helm installation.

serviceAccount.name → Uses the same ServiceAccount name linked to your Pod Identity association.

clusterName → Specifies the name of your EKS cluster.

vpcId → Supplies the EKS cluster’s VPC ID manually (required when IMDS auto-detection is restricted).

region → Explicitly sets the AWS Region to help the controller locate cluster and network resources when IMDS access is limited.


  
