# poc-karpenter

Proof of concept of the AWS Karpenter cluster autoscaler

# Installation methods

## Creating an EKS cluster with Karpenter enabled

Eksctl has [built-in support for Karpenter](https://eksctl.io/usage/eksctl-karpenter/). Under the hood a helm chart is being called.

```bash
eksctl create cluster -f deploy/eksctl/cluster-with-karpenter.yaml
```

> **WARNING**
>
>  Open issue
>  https://github.com/eksctl-io/eksctl/issues/7481
>  

And create a provisioner

```bash
#
# Get cluster credentials
#
eksctl utils write-kubeconfig --cluster cluster-with-karpenter-1

#
# Create a Karpenter provisioner
#
kubectl -n karpenter apply -f - <<END
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  limits:
    resources:
      cpu: 1000
  provider:
    instanceProfile: eksctl-KarpenterNodeInstanceProfile-cluster-with-karpenter-1
    subnetSelector:
      karpenter.sh/discovery: cluster-with-karpenter-1 # must match the tag set in the config file
    securityGroupSelector:
      karpenter.sh/discovery: cluster-with-karpenter-1 # must match the tag set in the config file
  ttlSecondsAfterEmpty: 30
END
```

### Cleanup

```bash
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```

## Using Helm chart to install Karpenter

The [Getting started](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/) guide describes how
to install Karpenter separately using helm.

### Set environment variables

```bash
export KARPENTER_NAMESPACE=kube-system
#export KARPENTER_VERSION=v0.33.1
export KARPENTER_VERSION=v0.32.5   # Stable release
export K8S_VERSION=1.28

export AWS_PARTITION="aws" 
export CLUSTER_NAME="cluster-with-karpenter-2"
export AWS_DEFAULT_REGION="eu-west-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT=$(mktemp)
```

### Provision EKS cluster

```bash
eksctl create cluster -f - <<EOF
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_DEFAULT_REGION}
  version: "${K8S_VERSION}"
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}

iam:
  withOIDC: true
  podIdentityAssociations:
   - namespace: "${KARPENTER_NAMESPACE}"
     serviceAccountName: karpenter
     roleName: ${CLUSTER_NAME}-karpenter
     permissionPolicyARNs:
     - arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME}

iamIdentityMappings:
- arn: "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}"
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes

managedNodeGroups:
- instanceType: m5.large
  amiFamily: AmazonLinux2
  name: ${CLUSTER_NAME}-ng
  desiredCapacity: 2
  minSize: 1
  maxSize: 10

addons:
 - name: eks-pod-identity-agent
EOF
```

Retrieve some cluster properties

```bash
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
export KARPENTER_IAM_ROLE_ARN="arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"

echo $CLUSTER_ENDPOINT $KARPENTER_IAM_ROLE_ARN
```

```bash
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
```

### Install Karpenter

```bash
#
# Get cluster credentials
#
eksctl utils write-kubeconfig --cluster $CLUSTER_NAME

# Logout of helm registry to perform an unauthenticated pull against the public ECR
helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" --create-namespace \
  --set "settings.clusterName=${CLUSTER_NAME}" \
  --set "settings.interruptionQueue=${CLUSTER_NAME}" \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --wait
```

### Create NodePool

```bash
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # 30 * 24h = 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
EOF
```

### Cleanup

```bash
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-2
```
