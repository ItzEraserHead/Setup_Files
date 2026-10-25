AWS EC2 DevOps Environment Setup Guide

Operating System: Amazon Linux 2023
Architecture: x86_64 (AMD64)
AWS Region: ap-south-1
Default AWS CLI Profile: eks-admin
Purpose: Install and configure AWS CLI, Terraform, Helm, kubectl, kubectx, kubens, and EKS connectivity on an EC2 instance.

Table of Contents
Prerequisites and configuration
Update the server and install dependencies
Install AWS CLI
Install Terraform
Install Helm
Install kubectl
Install kubectx and kubens
Configure AWS authentication
Manage AWS CLI profiles
Verify AWS identity and permissions
Discover EKS clusters
Configure kubectl for EKS
Verify Kubernetes connectivity
Switch Kubernetes contexts and namespaces
Troubleshooting
Security best practices
Final verification checklist
1. Prerequisites and Configuration

This guide prepares an Amazon Linux 2023 EC2 instance for AWS administration and Kubernetes operations.

Before starting, ensure you have:

An EC2 instance running Amazon Linux 2023.
SSH access and a user account with sudo privileges.
Network access to the required download sources and AWS endpoints.
An AWS identity with the necessary permissions.
An existing EKS cluster if you intend to test Kubernetes connectivity.
Configuration values

Customize these values for your environment.

Setting	Example
AWS Region	ap-south-1
AWS CLI Profile	eks-admin
EKS Cluster	my-cluster
Kubernetes Namespace	default
Architecture	x86_64

Check the operating system and architecture:

bash
cat /etc/os-release
uname -m


Expected architecture:

text
x86_64


Set the region and profile variables for the current shell:

bash
export AWS_REGION=ap-south-1
export AWS_PROFILE=eks-admin


These commands set environment variables only; they do not create a profile or configure credentials.

2. Update the Server and Install Dependencies

Update installed packages:

bash
sudo dnf update -y
sudo dnf install -y git unzip curl tar gzip


Verify the basic tools:

bash
git --version
curl --version
unzip -v

3. Install AWS CLI

Check whether AWS CLI is already installed:

bash
aws --version


If AWS CLI v2 is installed, continue to the authentication section.

If it is missing, install AWS CLI v2 for Linux AMD64 using the official AWS installation package:

bash
cd /tmp

curl -fsSLO \
  "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"

unzip -q awscli-exe-linux-x86_64.zip

sudo ./aws/install


If an existing installation needs updating, use the documented update procedure instead of repeatedly installing over it.

Verify:

bash
aws --version
which aws


The version output should identify AWS CLI version 2.

Official documentation:

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

4. Install Terraform

Add the official HashiCorp RPM repository:

bash
sudo dnf install -y dnf-plugins-core

sudo dnf config-manager --add-repo \
  https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo


Install Terraform:

bash
sudo dnf install -y terraform


Verify:

bash
terraform version


Optional: enable shell autocomplete in Bash:

bash
terraform -install-autocomplete


Restart the shell or start a new Bash session for autocomplete to take effect.

Official documentation:

https://developer.hashicorp.com/terraform/install

5. Install Helm

Use the official Helm installation script.

For production systems, review the script before executing it:

bash
curl -fsSLo /tmp/get-helm-3.sh \
  https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

less /tmp/get-helm-3.sh


After reviewing the script, execute it:

bash
bash /tmp/get-helm-3.sh


Verify:

bash
helm version


Helm is a Kubernetes package manager. Installing it does not grant access to a Kubernetes cluster.

Official documentation:

https://helm.sh/docs/intro/install/

6. Install kubectl

Download the latest stable kubectl client for Linux AMD64.

bash
cd /tmp

KUBECTL_VERSION=$(curl -fsSL \
  https://dl.k8s.io/release/stable.txt)

curl -fsSLO \
  "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl"

curl -fsSLO \
  "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl.sha256"


Verify the downloaded binary:

bash
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check


Install:

bash
sudo install -o root -g root -m 0755 \
  kubectl /usr/local/bin/kubectl


Verify:

bash
kubectl version --client
command -v kubectl


Compatibility note: When connecting to EKS, use a kubectl version compatible with the Kubernetes cluster version. The latest stable client is not automatically the best choice for every existing cluster.

Official documentation:

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

7. Install kubectx and kubens

Clone the official repository:

bash
sudo git clone \
  https://github.com/ahmetb/kubectx.git \
  /opt/kubectx


Create executable links:

bash
sudo ln -sf /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -sf /opt/kubectx/kubens /usr/local/bin/kubens


Verify:

bash
kubectx -h
kubens -h


Commands:

bash
kubectx
kubens

kubectx lists or switches Kubernetes contexts.
kubens lists or switches Kubernetes namespaces.

These tools change your local Kubernetes selection. They do not grant additional AWS or Kubernetes permissions.

Repository:

https://github.com/ahmetb/kubectx

8. Configure AWS Authentication

AWS CLI authentication can be configured in different ways. Select the approach appropriate for your EC2 instance.

Option A: EC2 instance has an attached IAM role (recommended)

Use an IAM role when the EC2 instance needs AWS access.

Create or select an IAM role with the required permissions.
Configure an EC2 instance profile for the role, if needed.
Attach the instance profile to the EC2 instance.
Verify that the AWS CLI can retrieve credentials.

Run:

bash
aws sts get-caller-identity


If AWS_PROFILE is set, the command may use that named profile instead of the instance role. To test the default credential chain without that profile selected:

bash
env -u AWS_PROFILE aws sts get-caller-identity


If necessary, inspect the effective configuration:

bash
env -u AWS_PROFILE aws configure list


The identity returned by STS should correspond to the expected IAM role.

Important: An EC2 IAM role does not require you to store IAM user access keys on the instance. Prefer temporary role credentials whenever possible.

Option B: EC2 instance has no attached IAM role

If the instance does not have an IAM role, you can configure an AWS CLI profile using an approved authentication method.

For an existing IAM user's access keys:

bash
aws configure --profile eks-admin


Enter the access key and secret access key when prompted.

Set the following values:

text
Default region name: ap-south-1
Default output format: json


Verify the profile:

bash
aws configure list --profile eks-admin
aws sts get-caller-identity --profile eks-admin


Never paste credentials into a shell command or commit them to a Git repository.

Production recommendation: Prefer an attached EC2 IAM role, IAM Identity Center, or another supported short-lived credential mechanism over long-lived IAM user keys.

9. Manage AWS CLI Profiles
List profiles
bash
aws configure list-profiles

Inspect a profile
bash
aws configure list --profile eks-admin


This displays the effective configuration and credential source information. It does not reveal the secret access key.

Use a profile for one command
bash
aws sts get-caller-identity --profile eks-admin


This is a good option when you do not want to change the current shell's default AWS profile.

Select a profile for the current terminal session
bash
export AWS_PROFILE=eks-admin
export AWS_REGION=ap-south-1


Verify:

bash
echo "$AWS_PROFILE"
echo "$AWS_REGION"

aws configure list
aws sts get-caller-identity

Persist the profile for future Bash sessions

Only do this if you deliberately want this profile to be the default for every new Bash session:

bash
echo 'export AWS_PROFILE=eks-admin' >> ~/.bashrc
echo 'export AWS_REGION=ap-south-1' >> ~/.bashrc

source ~/.bashrc


Avoid adding the same lines repeatedly.

Switch profiles
bash
export AWS_PROFILE=eks-admin


To use the default credential chain instead:

bash
unset AWS_PROFILE


Unsetting AWS_PROFILE does not guarantee that an EC2 role will be used. Other environment variables, explicit credential configuration, or other credential sources may still take precedence.

Understand AWS credential precedence

The AWS CLI may obtain credentials from environment variables, command-line options, named profiles, IAM Identity Center, role assumption, the EC2 instance metadata service, and other supported providers.

An explicitly selected profile can change which credentials are used.

If authentication behaves unexpectedly, inspect:

bash
env | grep '^AWS_' | sed -E \
  's/^(AWS_[A-Z0-9_]*(KEY|TOKEN|SECRET)[A-Z0-9_]*)=.*/\1=[REDACTED]/'


Avoid printing or sharing raw environment variables that may contain secrets.

AWS CLI configuration documentation:

https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html

10. Verify AWS Identity and Permissions

Check the identity being used:

bash
aws sts get-caller-identity


Or explicitly select the profile:

bash
aws sts get-caller-identity --profile eks-admin


The response contains the AWS account ID, ARN, and user ID.

Check the effective region and credential source:

bash
aws configure list
aws configure list --profile eks-admin


If a command fails with AccessDenied, verify that the selected identity has the necessary IAM permissions. For EKS, also verify the cluster's access configuration.

Authentication success does not imply authorization for every AWS service.

11. Discover EKS Clusters

List clusters in the configured region:

bash
aws eks list-clusters --region ap-south-1


Using an explicit profile:

bash
aws eks list-clusters \
  --region ap-south-1 \
  --profile eks-admin


To see the account being queried:

bash
aws sts get-caller-identity


If no clusters appear, check:

The selected AWS account.
The configured AWS region.
Whether the cluster exists.
Whether the selected identity has permission to list EKS clusters.

An empty cluster list does not necessarily mean that AWS authentication failed.

12. Configure kubectl for EKS

Replace my-cluster with the actual cluster name.

bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name my-cluster \
  --profile eks-admin \
  --alias my-cluster


This creates or updates the kubeconfig file, normally located at:

text
~/.kube/config


Inspect the configuration:

bash
kubectl config get-contexts
kubectl config current-context

Important authentication detail

aws eks update-kubeconfig configures kubectl to use AWS authentication for EKS. Depending on the setup, kubectl may invoke AWS CLI to obtain authentication tokens when commands run.

Ensure the credentials used by that process are valid. If you switch AWS profiles later, verify which identity kubectl will use.

Also ensure the selected IAM principal is authorized to access the EKS cluster through the cluster's access entries or other configured access mechanism.

13. Verify Kubernetes Connectivity

Check the active context:

bash
kubectl config current-context


Check cluster nodes:

bash
kubectl get nodes


List pods across namespaces:

bash
kubectl get pods -A


List namespaces:

bash
kubectl get namespaces


If these commands succeed, the current identity and network path allow the requested Kubernetes operations.

Remember that Kubernetes permissions may restrict access to some resources even when cluster connectivity works.

14. Switch Kubernetes Contexts and Namespaces
List contexts
bash
kubectx

Switch context
bash
kubectx my-cluster

Switch namespace
bash
kubens kube-system

Return to the previous context or namespace
bash
kubectx -
kubens -

Use kubectl without changing the current namespace
bash
kubectl get pods -n kube-system


A Kubernetes context selects a cluster, user, and optionally a default namespace. Switching contexts does not switch the AWS CLI profile automatically in every configuration.

For multiple AWS accounts or EKS clusters, verify both the AWS identity and Kubernetes context before performing administrative operations.

15. Troubleshooting
AWS CLI: Unable to locate credentials

Check:

bash
aws configure list
aws configure list-profiles
env | grep '^AWS_PROFILE='


For an IAM-role-based EC2 instance, verify that the correct role is attached and that the instance can retrieve credentials through the instance metadata service.

AWS CLI: AccessDenied

Check the active identity:

bash
aws sts get-caller-identity


Verify that the identity has the required service permissions.

EKS: Cluster not found

Check:

bash
aws sts get-caller-identity
aws eks list-clusters --region ap-south-1


Verify the cluster name, region, and AWS account.

kubectl: Unauthorized or Forbidden

Check:

bash
kubectl config current-context
aws sts get-caller-identity


Verify the EKS authentication configuration and the Kubernetes permissions granted to the identity.

kubectl: Connection timeout

Check:

Whether the EKS API endpoint is public or private.
Whether the EC2 instance has the required network route.
Security groups, network ACLs, and firewall rules.
DNS resolution and endpoint reachability.

A private EKS endpoint may require the EC2 instance to be inside the appropriate VPC or connected network.

kubectl: Wrong cluster

Run:

bash
kubectl config get-contexts
kubectl config current-context


Select the intended context before making changes.

Terraform: AWS authentication failure

Check the identity and profile:

bash
aws sts get-caller-identity
aws configure list


Confirm that Terraform is configured to use the intended credentials and region. Terraform's AWS provider configuration can override or differ from the AWS CLI defaults.

16. Security Best Practices
Prefer an EC2 IAM role over long-lived IAM user access keys.
Apply least-privilege IAM policies.
Use temporary credentials whenever practical.
Never commit AWS credentials, private keys, or sensitive Terraform state to Git.
Restrict access to ~/.aws and ~/.kube configuration files.
Treat kubeconfig files and Terraform state as potentially sensitive.
Avoid embedding credentials in Terraform files, shell scripts, or command history.
Verify the AWS account and Kubernetes context before running destructive operations.
Keep the operating system and installed tools updated.
Review downloaded installation scripts and verify binaries where supported.
Use appropriate logging and auditing for administrative activity.

Example permissions for local AWS configuration directories:

bash
chmod 700 ~/.aws
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config


Run these commands only for files you own and intend to secure. If a file does not exist, the corresponding command may report an error.

17. Final Verification Checklist
Installed tools
bash
git --version
aws --version
terraform version
helm version
kubectl version --client
kubectx -h
kubens -h

AWS authentication
bash
aws sts get-caller-identity
aws configure list
aws configure list-profiles


For a named profile:

bash
aws sts get-caller-identity --profile eks-admin

EKS access
bash
aws eks list-clusters \
  --region ap-south-1

kubectl config get-contexts
kubectl config current-context
kubectl get nodes
kubectl get pods -A

Completion criteria
Required tools are installed and executable.
The intended AWS identity is active.
The AWS account and region are correct.
EKS cluster discovery works.
The intended Kubernetes context is selected.
Kubernetes API access works.
Required permissions are confirmed.
Credentials and configuration files are handled securely.
Reference Documentation
AWS CLI installation: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
AWS CLI profiles: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
EC2 IAM roles: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
EKS kubeconfig: https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html
Terraform installation: https://developer.hashicorp.com/terraform/install
Helm installation: https://helm.sh/docs/intro/install/
kubectl installation: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
kubectx and kubens: https://github.com/ahmetb/kubectx

End of guide
