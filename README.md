# poc-karpenter

Proof of concept of the AWS Karpenter cluster autoscaler

# Creating an EKS cluster with Karpenter enabled

[Documentation](https://eksctl.io/usage/eksctl-karpenter/)

```
eksctl create cluster -f cluster-with-karpenter-v1.yaml
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

## Cleanup

```
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```



