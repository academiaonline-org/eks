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

