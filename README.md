# poc-karpenter

Proof of concept of the AWS Karpenter cluster autoscaler

# Creating an EKS cluster with Karpenter enabled

```
eksctl create cluster -f cluster-with-karpenter-v1.yaml
```

> **WARNING**
>
>  Open issue
>  https://github.com/eksctl-io/eksctl/issues/7481
>  

## Cleanup

```
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```
