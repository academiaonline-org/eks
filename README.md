# eks

All the steps will be performed from AWS Cloud9.

Install Terraform:
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
terraform version

```
Now you can clone the remote repository:
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

