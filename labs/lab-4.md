In this section, we will cover the essential aspects of working with an EKS cluster. We will accomplish the following tasks:
1. Expose the sample application to make it accessible over the public Internet.
2. Configure the worker nodes in the managed node group that powers the EKS cluster.
3. Enable and redeploy an application using Fargate, a serverless compute engine for containers.
4. Configure EBS (Elastic Block Store) and EFS (Elastic File System) for stateful applications.

By completing these tasks, you will gain a solid understanding of working with an EKS cluster and be equipped to leverage its features effectively.

Before you begin this section, make sure to prepare your environment by following these steps:
```
$ reset-environment
Resetting the environment, please wait
Waiting for application to become ready...
Environment is reset
```
In the previous lab, we deployed our sample application to EKS and observed the running Pods. However, you might be wondering where these Pods are actually running.

An EKS cluster consists of one or more EC2 nodes that serve as the underlying infrastructure for running Pods. These EKS nodes run within your AWS account and establish a connection to the control plane of your cluster through the cluster API server endpoint. To deploy these nodes, you create a node group, which is essentially an EC2 Auto Scaling group containing one or more EC2 instances.

It's important to note that EKS nodes are regular Amazon EC2 instances, and you are billed for them based on the standard EC2 pricing. For more details on pricing, you can refer to the Amazon EC2 pricing documentation.

To simplify the provisioning and management of nodes for Amazon EKS clusters, Amazon provides managed node groups. These managed node groups automate various operational tasks, including the provisioning and lifecycle management of nodes. This streamlines activities such as rolling updates for new Amazon Machine Images (AMIs) or Kubernetes version deployments, making it easier to maintain and manage your EKS clusters:
* https://www.eksworkshop.com/assets/images/managed-node-groups-388df0c767b6e5c4d77c7f8bf6f30f89.png

There are several advantages to running Amazon EKS managed node groups, including:
1. Easy provisioning and management: You can create, update, or terminate nodes in your managed node group with a single operation using various tools such as the Amazon EKS console, eksctl, AWS CLI, AWS API, or infrastructure-as-code tools like AWS CloudFormation and Terraform.
2. Latest optimized AMIs: The nodes provisioned as part of a managed node group run using the latest Amazon EKS optimized Amazon Machine Images (AMIs). This ensures that your nodes have the most up-to-date optimizations and improvements for running Kubernetes workloads on EKS.
3. Automatic metadata tagging: Nodes provisioned in a managed node group are automatically tagged with metadata such as availability zones, CPU architecture, and instance type. This simplifies resource management and allows you to easily identify and categorize your nodes.
4. Graceful node updates and terminations: When performing updates or terminating nodes in a managed node group, the process is automatically handled in a graceful manner. Nodes are drained of running applications and workloads to ensure high availability and minimal disruption to your applications.
5. No additional costs: There are no additional costs associated with using Amazon EKS managed node groups. You only pay for the AWS resources provisioned, such as the EC2 instances and associated services.

These advantages make Amazon EKS managed node groups a convenient and cost-effective option for provisioning and managing the nodes in your EKS clusters.

You can inspect the details of the default managed node group that was pre-provisioned for you using the following command:
```
$ eksctl get nodegroup --cluster $EKS_CLUSTER_NAME --name $EKS_DEFAULT_MNG_NAME
CLUSTER         NODEGROUP                                       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE       IMAGE ID        ASG NAME                                                                                TYPE
eks-workshop    managed-ondemand-20230610163837883200000027     ACTIVE  2023-06-10T16:38:40Z    2               6               2                       m5.large   AL2_x86_64       eks-managed-ondemand-20230610163837883200000027-2ac45316-4734-5123-9494-43265ac89da3    managed
```
From the output of the previous command, you can gather several attributes of the managed node group:
1. Minimum, Maximum, and Desired Counts: The configuration specifies the minimum, maximum, and desired counts for the number of nodes in the group. These settings control the scaling behavior of the managed node group.
2. Instance Type: The instance type for the nodes in the managed node group is specified as "m5.large". This indicates the specific type and size of the EC2 instances used for the nodes.
3. AMI Type: The managed node group uses the "AL2_x86_64" EKS Amazon Machine Image (AMI) type. This refers to the specific AMI that is used as the base image for the EC2 instances in the node group. The AMI type ensures that the nodes have the necessary software and configurations for running with EKS.

These attributes provide important information about the configuration and characteristics of the managed node group, helping you understand the node capacity, instance type, and underlying AMI used in your EKS cluster.

To inspect the nodes and their placement in the availability zones within your managed node group, you can use the following command:
```
$ kubectl get no -o wide --label-columns topology.kubernetes.io/zone
NAME                                          STATUS   ROLES    AGE     VERSION                INTERNAL-IP    EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION                 CONTAINER-RUNTIME   ZONE
ip-10-42-10-130.ap-south-1.compute.internal   Ready    <none>   7h46m   v1.23.15-eks-49d8fe8   10.42.10.130   <none>        Amazon Linux 2   5.4.228-131.415.amzn2.x86_64   docker://20.10.17   ap-south-1a
ip-10-42-10-170.ap-south-1.compute.internal   Ready    <none>   7h46m   v1.23.15-eks-49d8fe8   10.42.10.170   <none>        Amazon Linux 2   5.4.228-131.415.amzn2.x86_64   docker://20.10.17   ap-south-1a
ip-10-42-11-136.ap-south-1.compute.internal   Ready    <none>   7h46m   v1.23.15-eks-49d8fe8   10.42.11.136   <none>        Amazon Linux 2   5.4.228-131.415.amzn2.x86_64   docker://20.10.17   ap-south-1b
```
If you see that the nodes are distributed over multiple subnets in various availability zones, it indicates that the managed node group has been configured for high availability. This distribution helps ensure that your applications running on the nodes are resilient to failures in a specific availability zone.

By distributing the nodes across multiple subnets in different availability zones, the managed node group provides fault tolerance and enables your applications to continue running even if a single availability zone becomes unavailable. This high availability configuration helps maintain the overall availability and reliability of your applications deployed on the EKS cluster.

The distribution of nodes across multiple availability zones is an important design consideration to ensure the resilience and availability of your Kubernetes workloads in production environments.

To update your managed node group configuration and add additional nodes using the Amazon EKS Console, follow these steps:

1. Go to the Amazon EKS console by navigating to https://console.aws.amazon.com/eks/home#/clusters.
2. Select your eks-workshop cluster.
3. Click on the "Compute" tab.
4. Select the specific node group that you want to edit.
5. Choose the "Edit" button.

On the "Edit node group" page, you will find the "Node group scaling configuration" section which includes settings for "Desired size," "Minimum size," and "Maximum size." To add an additional node, you can increase the values of "Minimum size" and "Desired size" from 2 to 3, or as per your requirements.

After making the necessary changes, scroll down and click on the "Save changes" button to update the managed node group configuration.

Please note that you can also use the `eksctl scale nodegroup` command to scale your node group from the command line if you prefer a CLI-based approach.

To retrieve the current nodegroup scaling configuration and view the minimum size, maximum size, and desired capacity of nodes using the eksctl command, you can use the following command:
```
$ eksctl get nodegroup --name $EKS_DEFAULT_MNG_NAME --cluster $EKS_CLUSTER_NAME
CLUSTER         NODEGROUP                                       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID ASG NAME                                                                         TYPE
eks-workshop    managed-ondemand-20230610163837883200000027     ACTIVE  2023-06-10T16:38:40Z    2               6               2                       m5.large        AL2_x86_64eks-managed-ondemand-20230610163837883200000027-2ac45316-4734-5123-9494-43265ac89da3    managed
```
This command will display the details of the specified node group, including the scaling configuration.

Once you have retrieved the current configuration, you can modify the scaling settings as needed. To change the minimum size, maximum size, and desired capacity of nodes, you can use the eksctl scale nodegroup command.

Here's an example command to scale the node group:
```
$ eksctl scale nodegroup --name $EKS_DEFAULT_MNG_NAME --cluster $EKS_CLUSTER_NAME --nodes 3 --nodes-min 3 --nodes-max 6
2023-06-11 22:24:44 [ℹ]  scaling nodegroup "managed-ondemand-20230610163837883200000027" in cluster eks-workshop
2023-06-11 22:24:44 [ℹ]  waiting for scaling of nodegroup "managed-ondemand-20230610163837883200000027" to complete
2023-06-11 22:25:15 [ℹ]  nodegroup successfully scaled
```
To update a node group using the AWS CLI, you can use the update-nodegroup-config command:
```
$ aws eks update-nodegroup-config --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_DEFAULT_MNG_NAME --scaling-config minSize=3,maxSize=6,desiredSize=3
{
    "update": {
        "id": "472c8022-d784-3e23-9ca9-7a11bf22ee2f",
        "status": "InProgress",
        "type": "ConfigUpdate",
        "params": [
            {
                "type": "MinSize",
                "value": "3"
            },
            {
                "type": "MaxSize",
                "value": "6"
            },
            {
                "type": "DesiredSize",
                "value": "3"
            }
        ],
        "createdAt": "2023-06-11T22:28:36.503000+00:00",
        "errors": []
    }
}
```
After making changes to the node group through the Amazon EKS Console or the eksctl command, it may take approximately 2-3 minutes for the node provisioning and configuration changes to be applied. Let's fetch the node group configuration again and examine the minimum size, maximum size, and desired capacity of nodes using the eksctl command provided below:
```
$ eksctl get nodegroup --name $EKS_DEFAULT_MNG_NAME --cluster $EKS_CLUSTER_NAME
CLUSTER         NODEGROUP                                       STATUS  CREATED                 MIN SIZE        MAX SIZE        DESIRED CAPACITY        INSTANCE TYPE   IMAGE ID ASG NAME                                                                         TYPE
eks-workshop    managed-ondemand-20230610163837883200000027     ACTIVE  2023-06-10T16:38:40Z    3               6               3                       m5.large        AL2_x86_64eks-managed-ondemand-20230610163837883200000027-2ac45316-4734-5123-9494-43265ac89da3    managed
```
You can also review the updated worker node count using the following command, which lists all nodes in our managed node group by applying a filter based on the label:
```
$ kubectl get no -l eks.amazonaws.com/nodegroup=$EKS_DEFAULT_MNG_NAME
NAME                                          STATUS   ROLES    AGE    VERSION
ip-10-42-10-170.ap-south-1.compute.internal   Ready    <none>   29h    v1.23.15-eks-49d8fe8
ip-10-42-11-136.ap-south-1.compute.internal   Ready    <none>   29h    v1.23.15-eks-49d8fe8
ip-10-42-12-113.ap-south-1.compute.internal   Ready    <none>   3m2s   v1.23.15-eks-49d8fe8
```
Notice that the node is currently in a NotReady status, which occurs when the new node is still in the process of joining the cluster. To monitor the progress and wait until all nodes report as Ready, we can use the kubectl wait command:
```
$ kubectl wait --for=condition=Ready no -l eks.amazonaws.com/nodegroup=$EKS_DEFAULT_MNG_NAME --timeout=300s
node/ip-10-42-10-170.ap-south-1.compute.internal condition met
node/ip-10-42-11-136.ap-south-1.compute.internal condition met
node/ip-10-42-12-113.ap-south-1.compute.internal condition met
```
