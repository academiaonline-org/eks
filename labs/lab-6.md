Taints are properties of nodes that discourage the scheduling of certain pods. Tolerations, on the other hand, are applied to pods to allow their scheduling onto nodes with matching taints. Taints and tolerations work together to ensure that pods are not scheduled on unsuitable nodes. While tolerations enable pods to be scheduled on nodes with matching taints, it is not a guarantee, and other Kubernetes concepts like node affinity or node selectors may need to be used to achieve the desired configuration.

Configuring taints on nodes is useful in scenarios where we need to ensure that only specific pods are scheduled on certain node groups with special hardware (such as attached GPUs), or when we want to dedicate entire node groups to a particular set of Kubernetes users.

In this lab exercise, we will learn how to configure taints for our managed node groups and how to set up our applications to utilize tainted nodes.

For the purpose of this exercise, we have provisioned a separate managed node group that currently has no nodes running. Use the following command to scale the node group up to 1 node:
```
$ eksctl scale nodegroup --name $EKS_TAINTED_MNG_NAME --cluster $EKS_CLUSTER_NAME --nodes 1 --nodes-min 0 --nodes-max 1
2023-06-11 23:09:33 [ℹ]  scaling nodegroup "managed-ondemand-tainted-20230610163837083200000023" in cluster eks-workshop
2023-06-11 23:09:33 [ℹ]  waiting for scaling of nodegroup "managed-ondemand-tainted-20230610163837083200000023" to complete
2023-06-11 23:11:00 [ℹ]  nodegroup successfully scaled
```
Now, let's check the status of the node group by running the following command:
```
$ eksctl get nodegroup --name $EKS_TAINTED_MNG_NAME --cluster $EKS_CLUSTER_NAME
CLUSTER         NODEGROUP                                               STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID  ASG NAME                                                                                        TYPE
eks-workshop    managed-ondemand-tainted-20230610163837083200000023     ACTIVE  2023-06-10T16:38:40Z    0               1               1                       m5.large        AL2_x86_64        eks-managed-ondemand-tainted-20230610163837083200000023-4ec45316-45a0-1430-6210-debd1e036562    managed
```
It will take approximately 2-3 minutes for the node to join the EKS cluster. Please wait until you see the following output from the command:
```
$ kubectl get no --label-columns eks.amazonaws.com/nodegroup --selector eks.amazonaws.com/nodegroup=$EKS_TAINTED_MNG_NAME
NAME                                         STATUS   ROLES    AGE   VERSION                NODEGROUP
ip-10-42-10-65.ap-south-1.compute.internal   Ready    <none>   81s   v1.23.15-eks-49d8fe8   managed-ondemand-tainted-20230610163837083200000023
```
The command shown above utilizes the `--selector` flag to query for all nodes that have a label of `eks.amazonaws.com/nodegroup` matching the name of our managed node group, `$EKS_TAINTED_MNG_NAME`. The `--label-columns` flag also allows us to display the value of the `eks.amazonaws.com/nodegroup` label in the node list.

Before configuring our taints, let's examine the current configuration of our node. Please note that the following command will list the details of all nodes that are part of our managed node group. In our lab, the managed node group consists of just one instance:
```
$ kubectl describe no --selector eks.amazonaws.com/nodegroup=$EKS_TAINTED_MNG_NAME
Name:               ip-10-42-10-65.ap-south-1.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=m5.large
                    beta.kubernetes.io/os=linux
                    blocker=acaf0e72cf9fe0ad11bb06e0fc612de9cf50054d
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=managed-ondemand-tainted-20230610163837083200000023
                    eks.amazonaws.com/nodegroup-image=ami-061cf6003172b41c0
                    failure-domain.beta.kubernetes.io/region=ap-south-1
                    failure-domain.beta.kubernetes.io/zone=ap-south-1a
                    k8s.io/cloud-provider-aws=b2c4991f4c3acb5b142be2a5d455731a
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-10-42-10-65.ap-south-1.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=m5.large
                    tainted=yes
                    topology.ebs.csi.aws.com/zone=ap-south-1a
                    topology.kubernetes.io/region=ap-south-1
                    topology.kubernetes.io/zone=ap-south-1a
                    vpc.amazonaws.com/has-trunk-attached=true
                    workshop-default=no
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.42.10.65
                    csi.volume.kubernetes.io/nodeid: {"ebs.csi.aws.com":"i-0e9e4ad7bd5448eb6","efs.csi.aws.com":"i-0e9e4ad7bd5448eb6"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 11 Jun 2023 23:11:34 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  ip-10-42-10-65.ap-south-1.compute.internal
  AcquireTime:     <unset>
  RenewTime:       Sun, 11 Jun 2023 23:14:27 +0000
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 11 Jun 2023 23:12:36 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 11 Jun 2023 23:12:36 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 11 Jun 2023 23:12:36 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 11 Jun 2023 23:12:36 +0000   Sun, 11 Jun 2023 23:12:05 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:   10.42.10.65
  Hostname:     ip-10-42-10-65.ap-south-1.compute.internal
  InternalDNS:  ip-10-42-10-65.ap-south-1.compute.internal
Capacity:
  attachable-volumes-aws-ebs:  25
  cpu:                         2
  ephemeral-storage:           52416492Ki
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      7845568Ki
  pods:                        110
  vpc.amazonaws.com/pod-eni:   9
Allocatable:
  attachable-volumes-aws-ebs:  25
  cpu:                         1930m
  ephemeral-storage:           47233297124
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      7155392Ki
  pods:                        110
  vpc.amazonaws.com/pod-eni:   9
System Info:
  Machine ID:                 ec287207ca04d376c549758209a6e405
  System UUID:                ec287207-ca04-d376-c549-758209a6e405
  Boot ID:                    d344173d-8d2b-4d55-9c49-385117a4233f
  Kernel Version:             5.4.228-131.415.amzn2.x86_64
  OS Image:                   Amazon Linux 2
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://20.10.17
  Kubelet Version:            v1.23.15-eks-49d8fe8
  Kube-Proxy Version:         v1.23.15-eks-49d8fe8
ProviderID:                   aws:///ap-south-1a/i-0e9e4ad7bd5448eb6
Non-terminated Pods:          (6 in total)
  Namespace                   Name                                       CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                       ------------  ----------  ---------------  -------------  ---
  aws-for-fluent-bit          aws-for-fluent-bit-jqpb6                   50m (2%)      0 (0%)      50Mi (0%)        250Mi (3%)     2m29s
  kube-system                 aws-node-b2nng                             25m (1%)      0 (0%)      0 (0%)           0 (0%)         3m
  kube-system                 ebs-csi-node-zslt6                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3m
  kube-system                 efs-csi-node-bqwrh                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3m
  kube-system                 kube-proxy-nqqvq                           100m (5%)     0 (0%)      0 (0%)           0 (0%)         3m
  kubecost                    kubecost-prometheus-node-exporter-n8st5    0 (0%)        0 (0%)      0 (0%)           0 (0%)         2m29s
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests   Limits
  --------                    --------   ------
  cpu                         175m (9%)  0 (0%)
  memory                      50Mi (0%)  250Mi (3%)
  ephemeral-storage           0 (0%)     0 (0%)
  hugepages-1Gi               0 (0%)     0 (0%)
  hugepages-2Mi               0 (0%)     0 (0%)
  attachable-volumes-aws-ebs  0          0
  vpc.amazonaws.com/pod-eni   0          0
Events:
  Type    Reason                   Age                  From        Message
  ----    ------                   ----                 ----        -------
  Normal  Starting                 2m50s                kube-proxy  
  Normal  Starting                 3m6s                 kubelet     Starting kubelet.
  Normal  NodeHasSufficientMemory  3m6s (x2 over 3m6s)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    3m6s (x2 over 3m6s)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasNoDiskPressure
  Normal  NodeHasSufficientPID     3m6s (x2 over 3m6s)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasSufficientPID
  Normal  NodeAllocatableEnforced  3m6s                 kubelet     Updated Node Allocatable limit across pods
  Normal  NodeReady                2m29s                kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeReady
```
A few important points to note:

EKS automatically adds certain labels to facilitate easier filtering, including labels for the OS type, managed node group name, instance type, and more. While EKS provides certain labels by default, AWS allows operators to configure their own set of custom labels at the managed node group level. This ensures that every node within a node group has consistent labels.

At the moment, there are no taints configured for the node we examined, as indicated by the "Taints: <none>" section.
  
While it is possible to taint nodes using the kubectl CLI, administrators would need to manually make this change every time the underlying node group scales up or down. To address this challenge, AWS provides support for adding labels and taints to managed node groups, ensuring that every node within the node group automatically has the associated labels and taints configured.

In the upcoming sections, we will explore how to add taints to our preconfigured managed node group, $EKS_TAINTED_MNG_NAME.

Let's begin by adding a taint to our managed node group using the following AWS CLI command:
```
$ aws eks update-nodegroup-config --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_TAINTED_MNG_NAME --taints "addOrUpdateTaints=[{key=frontend, value=true, effect=NO_EXECUTE}]"
{
    "update": {
        "id": "c8eb80aa-7ba1-36b2-a798-61899c3d64ac",
        "status": "InProgress",
        "type": "ConfigUpdate",
        "params": [
            {
                "type": "TaintsToAdd",
                "value": "[{\"effect\":\"NO_EXECUTE\",\"value\":\"true\",\"key\":\"frontend\"}]"
            }
        ],
        "createdAt": "2023-06-11T23:19:29.253000+00:00",
        "errors": []
    }
}
$ aws eks wait nodegroup-active --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_TAINTED_MNG_NAME
```
The addition, removal, or replacement of taints can be performed using the `aws eks update-nodegroup-config` CLI command to update the configuration of the managed node group. You can use the `addOrUpdateTaints` or `removeTaints` options along with a list of taints specified using the `--taints` command flag.

The provided command will add a new taint with the key "frontend", value "true", and effect "NO_EXECUTE". This ensures that pods cannot be scheduled on any nodes within the managed node group unless they have a corresponding toleration. Additionally, any existing pods without a matching toleration will be evicted.

Note: The eksctl CLI can also be used to configure taints on a managed node group. Refer to the documentation for more information.

The taint effect values supported by the managed node group configuration are as follows:

- NO_SCHEDULE: Corresponds to the Kubernetes NoSchedule taint effect. It repels all pods without a matching toleration, but does not evict already running pods from the nodes.
- NO_EXECUTE: Corresponds to the Kubernetes NoExecute taint effect. In addition to repelling newly scheduled pods, it also evicts running pods that do not have a matching toleration.
- PREFER_NO_SCHEDULE: Corresponds to the Kubernetes PreferNoSchedule taint effect. If possible, EKS avoids scheduling pods that do not tolerate this taint on the node.

To check if the taints have been correctly configured for the managed node group, you can use the following command:
```
$ aws eks describe-nodegroup --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_TAINTED_MNG_NAME | jq .nodegroup.taints
[
  {
    "key": "frontend",
    "value": "true",
    "effect": "NO_EXECUTE"
  }
]
```
Note that it may take a few minutes for the managed node group to be updated and for the labels and taints to propagate. If you don't see any taints configured or receive a null value, please wait a few minutes before running the above command again.

You can also verify the taint propagation using the kubectl CLI with the following command:
```
$ kubectl describe no --selector eks.amazonaws.com/nodegroup=$EKS_TAINTED_MNG_NAME | grep Taints
Taints:             frontend=true:NoExecute
```
Now that we have applied a taint to our managed node group, we need to configure our application to utilize this change. Specifically, we want to deploy the ui microservice only on nodes that are part of the tainted managed node group.

Before making any changes, let's check the current configuration for the UI pods. Note that these pods are managed by a deployment called "ui":
```
$ kubectl describe po -n ui --selector app.kubernetes.io/name=ui
Name:         ui-fddcf8d7b-w7dtj
Namespace:    ui
Priority:     0
Node:         ip-10-42-10-170.ap-south-1.compute.internal/10.42.10.170
Start Time:   Sun, 11 Jun 2023 00:19:54 +0000
Labels:       app.kubernetes.io/component=service
              app.kubernetes.io/created-by=eks-workshop
              app.kubernetes.io/instance=ui
              app.kubernetes.io/name=ui
              pod-template-hash=fddcf8d7b
Annotations:  kubernetes.io/psp: eks.privileged
              prometheus.io/path: /actuator/prometheus
              prometheus.io/port: 8080
              prometheus.io/scrape: true
Status:       Running
IP:           10.42.10.204
IPs:
  IP:           10.42.10.204
Controlled By:  ReplicaSet/ui-fddcf8d7b
Containers:
  ui:
    Container ID:   docker://d8b6366ff6a573fd94d603b57d8b798944bed1fa524012b11240abe93436d29e
    Image:          public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
    Image ID:       docker-pullable://public.ecr.aws/aws-containers/retail-store-sample-ui@sha256:bbb537de803407627de8e7e3639674ef078844657196ff528c0dbb948c3f08e9
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 11 Jun 2023 00:20:16 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  1536Mi
    Requests:
      cpu:     250m
      memory:  1536Mi
    Liveness:  http-get http://:8080/actuator/health/liveness delay=45s timeout=1s period=20s #success=1 #failure=3
    Environment Variables from:
      ui  ConfigMap  Optional: false
    Environment:
      JAVA_OPTS:  -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
    Mounts:
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-b6d49 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  kube-api-access-b6d49:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
```
As expected, the application is currently running on a non-tainted node. The associated pod is in a "Running" status, and it does not have any custom tolerations configured. It's worth noting that Kubernetes automatically adds tolerations for "node.kubernetes.io/not-ready" and "node.kubernetes.io/unreachable" with a tolerationSeconds of 300, unless explicitly set by you or a controller. These auto-added tolerations ensure that pods remain bound to nodes for 5 minutes after any of these issues are detected.

Now, let's update our "ui" deployment to bind its pods to our tainted managed node group. We have pre-configured the tainted managed node group with a label of "tainted=yes" that we can use with a nodeSelector. The following Kustomize patch describes the necessary changes to our deployment configuration to enable this setup: /workspace/modules/fundamentals/mng/taints/nodeselector-wo-toleration/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    spec:
      nodeSelector:
        tainted: yes
```
To apply the Kustomize changes, you can use the following command:
```
$ kubectl apply -k /workspace/modules/fundamentals/mng/taints/nodeselector-wo-toleration/
namespace/ui unchanged
serviceaccount/ui unchanged
configmap/ui unchanged
service/ui unchanged
deployment.apps/ui configured
```
To check the rollout status of our UI deployment after making the recent changes, you can use the following command:
```
$ kubectl -n ui rollout status --watch=false deploy/ui
Waiting for deployment "ui" rollout to finish: 1 old replicas are pending termination...
```

Since the deployment rollout seems to be stuck, let's investigate further by checking the status of the deployment and its associated pods. We can use the following commands to gather more information:
```
$ kubectl get po -n ui -l app.kubernetes.io/name=ui
NAME                  READY   STATUS    RESTARTS   AGE
ui-6858d8cb47-qplgz   0/1     Pending   0          2m29s
ui-fddcf8d7b-w7dtj    1/1     Running   0          23h
```
By investigating the individual pods under the "ui" namespace, we can observe that one pod is in a "Pending" state. To gather more information about the issue, we can dive deeper into the details of the pending pod. The following command will provide more insights:
```
$ podname=$(kubectl get po -n ui --field-selector=status.phase=Pending -o json | jq -r '.items[0].metadata.name') && kubectl describe po $podname -n ui
Name:           ui-6858d8cb47-qplgz
Namespace:      ui
Priority:       0
Node:           <none>
Labels:         app.kubernetes.io/component=service
                app.kubernetes.io/created-by=eks-workshop
                app.kubernetes.io/instance=ui
                app.kubernetes.io/name=ui
                pod-template-hash=6858d8cb47
Annotations:    kubernetes.io/psp: eks.privileged
                prometheus.io/path: /actuator/prometheus
                prometheus.io/port: 8080
                prometheus.io/scrape: true
Status:         Pending
IP:             
IPs:            <none>
Controlled By:  ReplicaSet/ui-6858d8cb47
Containers:
  ui:
    Image:      public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      memory:  1536Mi
    Requests:
      cpu:     250m
      memory:  1536Mi
    Liveness:  http-get http://:8080/actuator/health/liveness delay=45s timeout=1s period=20s #success=1 #failure=3
    Environment Variables from:
      ui  ConfigMap  Optional: false
    Environment:
      JAVA_OPTS:  -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
    Mounts:
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rm6tm (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  kube-api-access-rm6tm:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              tainted=yes
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  54s (x5 over 5m18s)  default-scheduler  0/5 nodes are available: 1 node(s) had taint {frontend: true}, that the pod didn't tolerate, 1 node(s) had taint {systemComponent: true}, that the pod didn't tolerate, 3 node(s) didn't match Pod's node affinity/selector.
```
The changes we made are reflected in the new configuration of the pending pod. We can see that we have specified a nodeSelector to pin the pod to any node with the "tainted=yes" label. However, this has introduced a new problem as the pod cannot be scheduled (PodScheduled False). To get a more useful explanation, we can check the events associated with the pod:
```
0/5 nodes are available: 1 node(s) had taint {frontend: true}, that the pod didn't tolerate, 1 node(s) had taint {systemComponent: true}, that the pod didn't tolerate, 3 node(s) didn't match Pod's node affinity/selector.
```
To fix this issue, we need to add a toleration to our deployment and associated pods, allowing them to tolerate the "frontend: true" taint. You can make the necessary changes using the following Kustomize patch: /workspace/modules/fundamentals/mng/taints/nodeselector-w-toleration/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui
spec:
  template:
    spec:
      tolerations:
      - key: "frontend"
        operator: "Exists"
        effect: "NoExecute"
      nodeSelector:
        tainted: yes
```
To apply the kustomization with the updated toleration, you can use the following command:
```
$ kubectl apply -k /workspace/modules/fundamentals/mng/taints/nodeselector-w-toleration/
namespace/ui unchanged
serviceaccount/ui unchanged
configmap/ui unchanged
service/ui unchanged
deployment.apps/ui configured
```
After examining the UI pod, we can confirm that the configuration has been modified to include the desired toleration (frontend=true:NoExecute). As a result, the pod has been successfully scheduled on the node that has the corresponding taint. To validate this, you can use the following commands:
```
$ kubectl get po -n ui -l app.kubernetes.io/name=ui
NAME                  READY   STATUS    RESTARTS   AGE
ui-5b5456b856-gr9sx   1/1     Running   0          4m41s
```
```
$ kubectl describe po -n ui -l app.kubernetes.io/name=ui                                                                                               
Name:         ui-5b5456b856-gr9sx
Namespace:    ui
Priority:     0
Node:         ip-10-42-10-65.ap-south-1.compute.internal/10.42.10.65
Start Time:   Sun, 11 Jun 2023 23:38:56 +0000
Labels:       app.kubernetes.io/component=service
              app.kubernetes.io/created-by=eks-workshop
              app.kubernetes.io/instance=ui
              app.kubernetes.io/name=ui
              pod-template-hash=5b5456b856
Annotations:  kubernetes.io/psp: eks.privileged
              prometheus.io/path: /actuator/prometheus
              prometheus.io/port: 8080
              prometheus.io/scrape: true
Status:       Running
IP:           10.42.10.177
IPs:
  IP:           10.42.10.177
Controlled By:  ReplicaSet/ui-5b5456b856
Containers:
  ui:
    Container ID:   docker://cc05fb09c0906eee9925dbfafd4160074ca04faf987f30e725d19e87ea6d2b99
    Image:          public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0
    Image ID:       docker-pullable://public.ecr.aws/aws-containers/retail-store-sample-ui@sha256:bbb537de803407627de8e7e3639674ef078844657196ff528c0dbb948c3f08e9
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 11 Jun 2023 23:39:17 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  1536Mi
    Requests:
      cpu:     250m
      memory:  1536Mi
    Liveness:  http-get http://:8080/actuator/health/liveness delay=45s timeout=1s period=20s #success=1 #failure=3
    Environment Variables from:
      ui  ConfigMap  Optional: false
    Environment:
      JAVA_OPTS:  -XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom
    Mounts:
      /tmp from tmp-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jb7p7 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  tmp-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     Memory
    SizeLimit:  <unset>
  kube-api-access-jb7p7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              tainted=yes
Tolerations:                 frontend:NoExecute op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m14s  default-scheduler  Successfully assigned ui/ui-5b5456b856-gr9sx to ip-10-42-10-65.ap-south-1.compute.internal
  Normal  Pulling    5m13s  kubelet            Pulling image "public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0"
  Normal  Pulled     4m57s  kubelet            Successfully pulled image "public.ecr.aws/aws-containers/retail-store-sample-ui:0.4.0" in 16.522107697s
  Normal  Created    4m53s  kubelet            Created container ui
  Normal  Started    4m53s  kubelet            Started container ui
```
```
$ kubectl describe no --selector tainted=yes
Name:               ip-10-42-10-65.ap-south-1.compute.internal
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=m5.large
                    beta.kubernetes.io/os=linux
                    blocker=acaf0e72cf9fe0ad11bb06e0fc612de9cf50054d
                    eks.amazonaws.com/capacityType=ON_DEMAND
                    eks.amazonaws.com/nodegroup=managed-ondemand-tainted-20230610163837083200000023
                    eks.amazonaws.com/nodegroup-image=ami-061cf6003172b41c0
                    failure-domain.beta.kubernetes.io/region=ap-south-1
                    failure-domain.beta.kubernetes.io/zone=ap-south-1a
                    k8s.io/cloud-provider-aws=b2c4991f4c3acb5b142be2a5d455731a
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=ip-10-42-10-65.ap-south-1.compute.internal
                    kubernetes.io/os=linux
                    node.kubernetes.io/instance-type=m5.large
                    tainted=yes
                    topology.ebs.csi.aws.com/zone=ap-south-1a
                    topology.kubernetes.io/region=ap-south-1
                    topology.kubernetes.io/zone=ap-south-1a
                    vpc.amazonaws.com/has-trunk-attached=true
                    workshop-default=no
Annotations:        alpha.kubernetes.io/provided-node-ip: 10.42.10.65
                    csi.volume.kubernetes.io/nodeid: {"ebs.csi.aws.com":"i-0e9e4ad7bd5448eb6","efs.csi.aws.com":"i-0e9e4ad7bd5448eb6"}
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 11 Jun 2023 23:11:34 +0000
Taints:             frontend=true:NoExecute
Unschedulable:      false
Lease:
  HolderIdentity:  ip-10-42-10-65.ap-south-1.compute.internal
  AcquireTime:     <unset>
  RenewTime:       Sun, 11 Jun 2023 23:45:17 +0000
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sun, 11 Jun 2023 23:44:34 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 11 Jun 2023 23:44:34 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 11 Jun 2023 23:44:34 +0000   Sun, 11 Jun 2023 23:11:28 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 11 Jun 2023 23:44:34 +0000   Sun, 11 Jun 2023 23:12:05 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:   10.42.10.65
  Hostname:     ip-10-42-10-65.ap-south-1.compute.internal
  InternalDNS:  ip-10-42-10-65.ap-south-1.compute.internal
Capacity:
  attachable-volumes-aws-ebs:  25
  cpu:                         2
  ephemeral-storage:           52416492Ki
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      7845568Ki
  pods:                        110
  vpc.amazonaws.com/pod-eni:   9
Allocatable:
  attachable-volumes-aws-ebs:  25
  cpu:                         1930m
  ephemeral-storage:           47233297124
  hugepages-1Gi:               0
  hugepages-2Mi:               0
  memory:                      7155392Ki
  pods:                        110
  vpc.amazonaws.com/pod-eni:   9
System Info:
  Machine ID:                 ec287207ca04d376c549758209a6e405
  System UUID:                ec287207-ca04-d376-c549-758209a6e405
  Boot ID:                    d344173d-8d2b-4d55-9c49-385117a4233f
  Kernel Version:             5.4.228-131.415.amzn2.x86_64
  OS Image:                   Amazon Linux 2
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://20.10.17
  Kubelet Version:            v1.23.15-eks-49d8fe8
  Kube-Proxy Version:         v1.23.15-eks-49d8fe8
ProviderID:                   aws:///ap-south-1a/i-0e9e4ad7bd5448eb6
Non-terminated Pods:          (5 in total)
  Namespace                   Name                   CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                   ------------  ----------  ---------------  -------------  ---
  kube-system                 aws-node-b2nng         25m (1%)      0 (0%)      0 (0%)           0 (0%)         33m
  kube-system                 ebs-csi-node-zslt6     0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
  kube-system                 efs-csi-node-bqwrh     0 (0%)        0 (0%)      0 (0%)           0 (0%)         33m
  kube-system                 kube-proxy-nqqvq       100m (5%)     0 (0%)      0 (0%)           0 (0%)         33m
  ui                          ui-5b5456b856-gr9sx    250m (12%)    0 (0%)      1536Mi (21%)     1536Mi (21%)   6m25s
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                    Requests      Limits
  --------                    --------      ------
  cpu                         375m (19%)    0 (0%)
  memory                      1536Mi (21%)  1536Mi (21%)
  ephemeral-storage           0 (0%)        0 (0%)
  hugepages-1Gi               0 (0%)        0 (0%)
  hugepages-2Mi               0 (0%)        0 (0%)
  attachable-volumes-aws-ebs  0             0
  vpc.amazonaws.com/pod-eni   0             0
Events:
  Type    Reason                   Age                From        Message
  ----    ------                   ----               ----        -------
  Normal  Starting                 33m                kube-proxy  
  Normal  Starting                 33m                kubelet     Starting kubelet.
  Normal  NodeHasSufficientMemory  33m (x2 over 33m)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    33m (x2 over 33m)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasNoDiskPressure
  Normal  NodeHasSufficientPID     33m (x2 over 33m)  kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeHasSufficientPID
  Normal  NodeAllocatableEnforced  33m                kubelet     Updated Node Allocatable limit across pods
  Normal  NodeReady                33m                kubelet     Node ip-10-42-10-65.ap-south-1.compute.internal status is now: NodeReady
```
