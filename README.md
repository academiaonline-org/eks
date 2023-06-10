# eks

All the steps will be performed from a Linux terminal (I am using an EC2 instance). You need to run a BASH shell. You can do it as root or as a normal user:
```
sudo su --login root

```
Remove unwanted aliases if necessary:
```
unalias rm cp mv

```
Install git and docker if not yet available:
```
yum install -y docker git
systemctl enable --now docker

```
Install Terraform:
```
wget https://releases.hashicorp.com/terraform/1.3.9/terraform_1.3.9_linux_amd64.zip
unzip terraform_1.3.9_linux_amd64.zip
install terraform /usr/local/bin/
terraform version

```
You need enough AWS privileges (AdministratorAccess will work). For this purpose you need to create a new Access Key in your AWS IAM Security Credentials and then configure the AWS credentials in your Linux terminal:
* https://console.aws.amazon.com/iam/home
```bash
aws configure

```
Now you can clone the remote repository:
```
git clone https://github.com/aws-samples/eks-workshop-v2.git
cd eks-workshop-v2
git checkout latest
cd terraform

```
To initiate Terraform and create the supporting infrastructure, execute the following command:
```bash
terraform init
terraform plan

```
When you run `terraform init`, it will initialize your Terraform working directory and download any necessary provider plugins. After running `terraform init`, you can run `terraform apply` to create the infrastructure defined in your Terraform configuration files.

Make sure that you review the plan before approving the `terraform apply` command as this will make changes to your infrastructure:
```
terraform apply --auto-approve

```

