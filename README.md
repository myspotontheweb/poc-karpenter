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
> 
> Pods are crashlooping
> 
> ```
> $ kubectl get all
> NAME                            READY   STATUS             RESTARTS         AGE
> pod/karpenter-b5475c9cc-jsgns   0/1     CrashLoopBackOff   15 (2m45s ago)   54m
> pod/karpenter-b5475c9cc-l9jrr   0/1     CrashLoopBackOff   15 (2m24s ago)   54m
> ```

## Cleanup

```
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```