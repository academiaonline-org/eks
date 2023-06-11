Pods can be assigned to specific nodes or have specific scheduling requirements based on various factors. This includes scenarios where you want only one application pod to run on each node or when you want pods to be co-located on the same node. Node affinity can be used to specify preferred or mandatory constraints for pod scheduling.

In this lesson, we will focus on inter-pod affinity and anti-affinity by configuring the checkout-redis pods to run only one instance per node and ensuring that the checkout pods run only on nodes where a checkout-redis pod exists. This will optimize performance by ensuring that the caching pods (checkout-redis) are colocated with a checkout pod instance.

First, let's verify that the checkout and checkout-redis pods are currently running:
```
$ kubectl get po -n checkout
NAME                              READY   STATUS    RESTARTS   AGE
checkout-f4954f4d7-4bjhn          1/1     Running   0          22h
checkout-redis-6cfd7d8787-vggkv   1/1     Running   0          23h
```
We can observe that both applications have one pod running in the cluster. Now, let's determine the nodes on which they are running:
```
$ kubectl get po -n checkout -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-f4954f4d7-4bjhn        ip-10-42-10-170.ap-south-1.compute.internal
checkout-redis-6cfd7d8787-vggkv ip-10-42-11-136.ap-south-1.compute.internal
```
According to the provided information, the "checkout-f4954f4d7-4bjhn" pod is running on the "ip-10-42-10-170.ap-south-1.compute.internal" node, and the "checkout-redis-6cfd7d8787-vggkv" pod is running on the "ip-10-42-11-136.ap-south-1.compute.internal" node.

Let's configure a podAffinity and podAntiAffinity policy in the checkout deployment to ensure that only one checkout pod runs per node and that it runs only on nodes where a checkout-redis pod is already running. We'll use the requiredDuringSchedulingIgnoredDuringExecution setting to enforce this requirement rather than making it a preferred behavior.

The following kustomization includes an affinity section in the checkout deployment with the podAffinity and podAntiAffinity policies specified: /workspace/modules/fundamentals/affinity/checkout/checkout.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
  namespace: checkout
spec:
  template:
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/component
                operator: In
                values:
                - redis
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/component
                operator: In
                values:
                - service
              - key: app.kubernetes.io/instance
                operator: In
                values:
                - checkout
            topologyKey: kubernetes.io/hostname
```
To apply the changes, execute the following command to update the checkout deployment in your cluster:
```
$ kubectl delete -n checkout deploy checkout
deployment.apps "checkout" deleted
$ kubectl apply -k /workspace/modules/fundamentals/affinity/checkout/
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout created
deployment.apps/checkout-redis unchanged
$ kubectl rollout status deploy/checkout -n checkout --timeout 180s
deployment "checkout" successfully rolled out
```
The podAffinity section ensures that a checkout-redis pod is running on the node since the checkout pod requires checkout-redis to function properly. On the other hand, the podAntiAffinity section ensures that no checkout pods are already running on the node, using the app.kubernetes.io/component=service label as a matching criteria. Now, let's increase the deployment's scale to verify that the configuration is functioning correctly.
```
$ kubectl scale -n checkout deploy/checkout --replicas 2
deployment.apps/checkout scaled
```
Now let's validate the placement of each pod:
```
$ kubectl get po -n checkout -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-7d5867f695-b5mgn       ip-10-42-11-136.ap-south-1.compute.internal
checkout-7d5867f695-j6j7j
checkout-redis-6cfd7d8787-vggkv ip-10-42-11-136.ap-south-1.compute.internal
```
In this example, the first checkout pod runs on the same node as the existing checkout-redis pod, as it satisfies the podAffinity rule we set. However, the second checkout pod remains in a pending state because the podAntiAffinity rule we defined prevents two checkout pods from running on the same node. Since the second node doesn't have a checkout-redis pod running, the pod remains pending.

Next, let's scale the checkout-redis deployment to two instances, each running on a different node. To achieve this, we will modify the checkout-redis deployment policy by adding a podAntiAffinity rule: /workspace/modules/fundamentals/affinity/checkout-redis/checkout-redis.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-redis
  labels:
    app.kubernetes.io/created-by: eks-workshop
    app.kubernetes.io/team: database
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/component
                operator: In
                values:
                - redis
            topologyKey: kubernetes.io/hostname
```
Apply the changes by running the following command:
```
$ kubectl delete -n checkout deployment checkout-redis
deployment.apps "checkout-redis" deleted
$ kubectl apply -k /workspace/modules/fundamentals/affinity/checkout-redis/
namespace/checkout unchanged
serviceaccount/checkout unchanged
configmap/checkout unchanged
service/checkout unchanged
service/checkout-redis unchanged
deployment.apps/checkout configured
deployment.apps/checkout-redis created
$ kubectl rollout status deploy/checkout-redis -n checkout --timeout 180s                                                                              
deployment "checkout-redis" successfully rolled out
```
The podAntiAffinity section ensures that there are no checkout-redis pods already running on the node by matching the app.kubernetes.io/component=redis label:
```
$ kubectl scale -n checkout deploy/checkout-redis --replicas 2
deployment.apps/checkout-redis scaled
```
Check the running pods to verify that there are now two instances of each pod running:
```
$ kubectl get po -n checkout
NAME                              READY   STATUS    RESTARTS   AGE
checkout-7d5867f695-b5mgn         1/1     Running   0          12m
checkout-7d5867f695-j6j7j         1/1     Running   0          7m53s
checkout-redis-5996754d4b-v7kj6   1/1     Running   0          3m4s
checkout-redis-5996754d4b-zp2mp   1/1     Running   0          61s
```
We can also verify the node placement of the pods to ensure that the podAffinity and podAntiAffinity policies are being enforced:
```
$ kubectl get po -n checkout -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}'
checkout-7d5867f695-b5mgn       ip-10-42-11-136.ap-south-1.compute.internal
checkout-7d5867f695-j6j7j       ip-10-42-12-113.ap-south-1.compute.internal
checkout-redis-5996754d4b-v7kj6 ip-10-42-11-136.ap-south-1.compute.internal
checkout-redis-5996754d4b-zp2mp ip-10-42-12-113.ap-south-1.compute.internal
```
All looks good regarding pod scheduling, but we can further verify by scaling the checkout pod again to see where a third pod will be deployed:
```
$ kubectl scale --replicas=3 deploy/checkout -n checkout
deployment.apps/checkout scaled
```
If we check the running pods, we can see that the third checkout pod is in a Pending state because two nodes already have a pod deployed and the third node does not have a checkout-redis pod running.
```
$ kubectl get po -n checkout
NAME                              READY   STATUS    RESTARTS   AGE
checkout-7d5867f695-b5mgn         1/1     Running   0          15m
checkout-7d5867f695-gm5wp         0/1     Pending   0          57s
checkout-7d5867f695-j6j7j         1/1     Running   0          11m
checkout-redis-5996754d4b-v7kj6   1/1     Running   0          6m18s
checkout-redis-5996754d4b-zp2mp   1/1     Running   0          4m15s
```
Let's conclude this section by removing the Pending pod:
```
$ kubectl scale --replicas=2 deploy/checkout -n checkout
deployment.apps/checkout scaled
$ kubectl get po -n checkout                                                                                               
NAME                              READY   STATUS    RESTARTS   AGE
checkout-7d5867f695-b5mgn         1/1     Running   0          17m
checkout-7d5867f695-j6j7j         1/1     Running   0          12m
checkout-redis-5996754d4b-v7kj6   1/1     Running   0          7m27s
checkout-redis-5996754d4b-zp2mp   1/1     Running   0          5m24s
```
