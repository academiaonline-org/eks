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
```
  
