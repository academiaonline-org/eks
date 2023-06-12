Our current compute nodes in the EKS cluster are utilizing On-Demand capacity, but there are alternative options available for EC2 customers, including Spot Instances.

Spot Instances are instances that utilize spare EC2 capacity and are available at a lower cost compared to On-Demand instances. By requesting unused EC2 instances at discounted prices, you can significantly reduce your Amazon EC2 costs. The price for a Spot Instance is known as the Spot price, which is determined by Amazon EC2 and adjusted based on supply and demand.

Spot Instances are particularly beneficial if you have flexibility in terms of when your applications can run and if your applications can handle interruptions. They are well-suited for use cases such as data analysis, batch jobs, background processing, and optional tasks. You can find more information about Spot Instances in the Amazon EC2 documentation.

In this lab exercise, we will explore how to provision Spot capacity for our EKS cluster and deploy workloads that make use of Spot Instances. By leveraging Spot capacity, you can optimize costs while still running your workloads effectively.

Amazon EKS managed node groups with Spot capacity enhance the experience of managing EC2 instances by providing easy provisioning and management of Spot Instances. With EKS managed node groups, you can launch an EC2 Auto Scaling group with Spot best practices and handle Spot Instance interruptions automatically. This allows you to take advantage of the significant cost savings offered by Spot Instances for containerized applications that can tolerate interruptions.

In addition to the benefits of managed node groups, EKS managed node groups with Spot capacity offer the following advantages:
1. Allocation strategy: The Spot capacity allocation strategy is set to Capacity Optimized, ensuring that Spot nodes are provisioned in the most optimal Spot capacity pools.
2. Multiple instance types: You can specify multiple instance types during the creation of managed node groups, increasing the number of Spot capacity pools available for allocating capacity.
3. Automatic tagging: Nodes provisioned under managed node groups with Spot capacity are automatically tagged with the capacity type: eks.amazonaws.com/capacityType: SPOT. This label can be used to schedule fault-tolerant applications on Spot nodes.
4. Spot Capacity Rebalancing: Amazon EC2 Spot Capacity Rebalancing is enabled, allowing Amazon EKS to gracefully drain and rebalance Spot nodes to minimize disruption to applications when a Spot node is at an elevated risk of interruption.

These features provide an efficient and reliable way to leverage Spot capacity within your EKS cluster, maximizing cost savings while maintaining application availability.

The EKS-Optimized Amazon Linux AMI is the recommended base image for Amazon EKS nodes. It is built on Amazon Linux 2 and regularly updated with Kubernetes patches and security updates. It is best practice to use the latest version of the EKS-Optimized AMI when adding nodes to your EKS cluster and upgrading existing nodes.

EKS managed node groups provide automated AMI update capability. The process of upgrading the AMI for managed node groups consists of four phases:

1. Setup: 
   - Create a new version of the Amazon EC2 Launch Template associated with the Auto Scaling group, using the latest AMI.
   - Point the Auto Scaling group to use the latest version of the launch template.
   - Determine the maximum number of nodes to upgrade in parallel by configuring the updateconfig property for the node group.

2. Scale Up: 
   - During the upgrade process, launch upgraded nodes in the same availability zone as the nodes being upgraded.
   - Increment the maximum size and desired size of the Auto Scaling group to accommodate the additional nodes.
   - Check if the nodes with the latest configuration are present in the node group.
   - Apply the eks.amazonaws.com/nodegroup=unschedulable:NoSchedule taint on nodes that do not have the latest labels, to prevent previously updated nodes from being tainted.

3. Upgrade: 
   - Select a node randomly and drain the Pods from that node.
   - Cordon the node after all Pods have been evicted and wait for 60 seconds.
   - Send a termination request to the Auto Scaling group for the cordoned node.
   - Apply the same process to all nodes in the managed node group to ensure there are no nodes with older versions.

4. Scale Down: 
   - Decrease the maximum size and desired size of the Auto Scaling group by one in each iteration until the values are the same as before the update started.

This managed node group update behavior ensures that the AMI for the nodes is automatically updated while respecting Pod disruption budgets and maintaining high availability for your applications. For more detailed information about managed node group update phases, you can refer to the official documentation.

The provided EKS cluster has intentionally deployed managed node groups using outdated AMIs. To check the latest AMI version, you can retrieve the information by querying the SSM service:
```
$ aws ssm get-parameter --name /aws/service/eks/optimized-ami/1.23/amazon-linux-2/recommended/image_id --region $AWS_DEFAULT_REGION --query "Parameter.Value" --output text
ami-0c0bc24b5c6ec2fbf
```
When you initiate an update for a managed node group, Amazon EKS takes care of automatically updating your nodes. This includes applying the latest security patches and operating system updates if you are using an Amazon EKS optimized AMI. To update the managed node group hosting our sample application, you can use the following command:
```
$ aws eks update-nodegroup-version --cluster-name $EKS_CLUSTER_NAME --nodegroup-name $EKS_DEFAULT_MNG_NAME
{
    "update": {
        "id": "edd40821-0430-3cf5-b65b-1ede7e7e620b",
        "status": "InProgress",
        "type": "VersionUpdate",
        "params": [
            {
                "type": "Version",
                "value": "1.23"
            },
            {
                "type": "ReleaseVersion",
                "value": "1.23.17-20230607"
            }
        ],
        "createdAt": "2023-06-12T00:15:07.759000+00:00",
        "errors": []
    }
}
```
