# Three-Tier E-Commerce Application Deployment on AWS EKS

The **Robot Shop** application, mimics real-world e-commerce platforms like Amazon or Flipkart, with features such as user registration, login, catalog browsing, cart management, and checkout.

## Architecture Overview

The application adheres to the **three-tier architecture** model:

- **Presentation Layer**:
   - User interface built with Angular.
   - Provides registration, login, and product browsing functionalities.

- **Logic Layer**:
   - Microservices handle business logic, such as catalog management, cart operations, and payment processing.
   - Written in various programming languages (Python, Go, PHP, etc.).

- **Data Layer**:
   - Databases and Redis for persisting and caching data### Data Layer

### Components

- **Microservices**: Catalog, ratings, cart, payments, shipping, user management.
- **Databases**:
  - MongoDB for user data storage.
  - MySQL for product catalog and ratings.
- **Redis**: Manages cart session data (In-Memory Data Store).
- **RabbitMQ**: Handles order queues.

## Deployment on AWS EKS

### 1. Prerequisites
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install Helm
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-add-repository "deb https://baltocdn.com/helm/stable/debian/ all main"
sudo apt update && sudo apt install -y helm
helm version
```

### 2. EKS Cluster Setup

```bash
# Create the EKS Cluster:
eksctl create cluster \
    --name demo-cluster-eks-robotshop \
    --region us-west-1 \
    --nodegroup-name standard-workers  \
    --node-type t3.medium \
    --nodes 2

# Verify the Cluster:
kubectl get nodes
```

### 3. Configure OIDC IAM Integration

```bash
# Export cluster name
export CLUSTER_NAME=demo-cluster-eks-robotshop

# Get OIDC ID
oidc_id=$(aws eks describe-cluster --name $CLUSTER_NAME --region us-west-1 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check if OIDC provider exists
aws iam list-open-id-connect-providers | grep $oidc_id

# Create OIDC provider if not exists
eksctl utils associate-iam-oidc-provider \
    --cluster $CLUSTER_NAME \
    --region us-west-1 \
    --approve
```

### 4. ALB Controller Setup

```bash
# Download IAM policy
curl -o iam_policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
  

# Create IAM Policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create IAM role
eksctl create iamserviceaccount \
    --cluster=$CLUSTER_NAME \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --role-name AmazonEKSLoadBalancerControllerRole \
    --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/    AWSLoadBalancerControllerIAMPolicy \
    --approve

# Deploy ALB controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=$CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    --set region=us-west-1 \
    --set vpcId=$VPC_ID

# Verify that the deployments are running.
kubectl get deployment -n kube-system aws-load-balancer-controller

# Create an Ingress Resource: Use Kubernetes Ingress to expose the application via the ALB.
```
```bash
# You might face the issue, unable to see the loadbalancer address while giving k get ing -n robot-shop at the end. To avoid this your **AWSLoadBalancerControllerIAMPolicy** should have the required permissions for elasticloadbalancing:DescribeListenerAttributes.

# Run the following command to retrieve the policy details and look for **elasticloadbalancing:DescribeListenerAttributes** in the policy document.
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text)

# If the required permission is missing, update the policy to include it
# Download the current policy
aws iam get-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --version-id $(aws iam get-policy --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy --query 'Policy.DefaultVersionId' --output text) \
    --query 'PolicyVersion.Document' --output json > policy.json

# Edit policy.json to add the missing permissions
{
  "Effect": "Allow",
  "Action": "elasticloadbalancing:DescribeListenerAttributes",
  "Resource": "*"
}

# Create a new policy version
aws iam create-policy-version \
    --policy-arn arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://policy.json \
    --set-as-default
```

### 5. Configure EBS CSI Driver

```bash
# The Amazon EBS CSI plugin requires IAM permissions to make calls to AWS APIs on your behalf.

# Create an IAM role and attach a policy. AWS maintains an AWS managed policy or you can create your own custom policy. You can create an IAM role and attach the AWS managed policy with the following command. The command deploys an AWS CloudFormation stack that creates an IAM role and attaches the IAM policy to it. 

# Create IAM role for EBS CSI (AWS managed policy)
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve

# If you choose to create your own custom policy, skip the above command 
# # Create IAM policy
# curl -o ebs_csi_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
# aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs_csi_policy.json
# # Create IAM role
# eksctl create iamserviceaccount \
#     --name ebs-csi-controller-sa \
#     --namespace kube-system \
#     --cluster $CLUSTER_NAME \
#     --attach-policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
#     --approve
```
```bash
# Add EBS CSI add-on
# This command is used to add the AWS EBS CSI driver as a managed EKS add-on. It leverages Amazon EKS's built-in integration with the CSI driver and is simpler to manage because EKS handles updates and configurations for the add-on.
eksctl create addon \
    --name aws-ebs-csi-driver \
    --cluster $CLUSTER_NAME \
    --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/   AmazonEKS_EBS_CSI_DriverRole \
    --force

# Install EBS CSI driver using helm
# This command uses Helm to install the AWS EBS CSI driver manually. It gives you more flexibility and control over the installation, allowing for custom configurations such as custom namespace, specific versions, or additional Helm chart options.
# helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
# helm repo update
# helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
#     --namespace kube-system \
#     --set controller.serviceAccount.create=false \
#     --set controller.serviceAccount.name=ebs-csi-controller-sa

# Note: If your cluster is in the AWS GovCloud (US-East) or AWS GovCloud (US-West) AWS Regions, then replace arn:aws: with arn:aws-us-gov:.

# References: https://repost.aws/knowledge-center/eks-persistent-storage
```

### 6. Deploy Application

The application is managed using a **Helm chart**, simplifying deployment and configuration management.

```bash
# Create namespace
kubectl create namespace robot-shop

# Deploy using Helm
cd EKS/helm
helm install robot-shop --namespace robot-shop .

# Apply the Ingress configuration:
kubectl apply -f ingress.yaml -n robot-shop
```

### 7. Verify Deployment

```bash
# Check pod status
kubectl get pods -n robot-shop

# Check services
kubectl get svc -n robot-shop

# Retrieve the ALB Ingress endpoint:
kubectl get ingress -n robot-shop

# Visit the URL in your browser to explore the Robot Shop interface.
```

### 8. Cleanup

```bash
# To avoid incurring unnecessary costs, delete the EKS cluster and associated resources after testing:

helm uninstall robot-shop -n robot-shop

eksctl delete cluster --name $CLUSTER_NAME --region us-west-1
```

### 9. Troubleshooting

- **Check pod logs:**
    `kubectl logs <pod-name> -n robot-shop`

- **ALB Issues:**
    - Verify ALB controller pods are running
    - Check ALB security groups
    - Review ALB controller logs

- **Storage Issues:**
    - Verify EBS CSI driver status
    - Check PVC/PV binding status
    - Review storage class configuration

## Best Practices

- **Separate Microservices**: Use individual Helm charts for each microservice in production environments.
- **Secure Resource Access**: Assign unique IAM roles for microservices needing AWS access.
- **Monitoring and Logging**: Implement tools like Prometheus and Grafana.
- **CI/CD Pipelines**: Automate deployments using CI/CD pipelines for each microservice.

## References

- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Kubernetes Helm Charts](https://helm.sh/docs/)
- [Redis on Kubernetes](https://redis.io/docs/getting-started/)
- [ALB Ingress Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
