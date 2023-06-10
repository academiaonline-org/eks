# eks
Workshop based on the following:
* https://www.eksworkshop.com/docs/introduction/

All the steps will be performed from AWS Cloud9. 
You will need an account with enough AWS privileges (AdministratorAccess will work).
For this purpose you need to create a new Access Key in your AWS IAM Security Credentials and then configure the AWS credentials in your Linux terminal:
* https://console.aws.amazon.com/iam/home
```bash
aws configure

```
Clone the remote repository:
```
git clone https://github.com/aws-samples/eks-workshop-v2.git
cd eks-workshop-v2
git checkout latest
cd terraform
sed -i s/3.19.0/5.0.0/ modules/cluster/vpc.tf

```
To initiate Terraform and create the supporting infrastructure, execute the following command:
```bash
terraform init
terraform plan

```
Make sure that you review the plan before approving the `terraform apply` command as this will make changes to your infrastructure:
```
terraform apply --auto-approve

```
If after running the Terraform script you do not see the Cloud9 instance named eks-workshop do the following:
* https://www.eksworkshop.com/docs/misc/cloud9-access
