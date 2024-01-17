# poc-karpenter

Proof of concept of the AWS Karpenter cluster autoscaler

# Creating an EKS cluster with Karpenter enabled

```
eksctl create cluster -f cluster-with-karpenter-v1.yaml
```

> **WARNING**
> 
> Command not currently working, getting the following error:
> 
>  ```failed to install Karpenter chart: failed to install chart: timed out waiting for the condition```

## Cleanup

```
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```