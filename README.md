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
>
> Chasing root cause of this message
> 
> ```
> $ kubectl logs karpenter-b5475c9cc-jsgns
> panic: validating options, missing field, cluster-name
> 
> goroutine 1 [running]:
> github.com/samber/lo.must({0x25cbe60, 0xc00074c4c0}, {0x0, 0x0, 0x0})
>         github.com/samber/lo@v1.39.0/errors.go:53 +0x1e9
> github.com/samber/lo.Must0(...)
>         github.com/samber/lo@v1.39.0/errors.go:72
> sigs.k8s.io/karpenter/pkg/operator/injection.WithOptionsOrDie({0x30c0598, 0xc0003290b0}, {0xc0000c39a0, > 0x2, 0x2333e00?})
>         sigs.k8s.io/karpenter@v0.33.1/pkg/operator/injection/injection.go:51 +0x138
> sigs.k8s.io/karpenter/pkg/operator.NewOperator()
>         sigs.k8s.io/karpenter@v0.33.1/pkg/operator/operator.go:84 +0xb7
> main.main()
>         github.com/aws/karpenter/cmd/controller/main.go:33 +0x25
> ```

## Cleanup

```
eksctl delete cluster --region=eu-west-1 --name=cluster-with-karpenter-1
```
