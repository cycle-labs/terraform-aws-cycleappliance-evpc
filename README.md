# Cycle Appliance 
[[_TOC_]]

# Cycle Appliance Overview

The Cycle Appliance consolidates all the essential cloud infrastructure required to seamlessly execute Cycle tests autonomously. Instead of grappling with the complexities of setting up Cycle on a local machine or manually configuring cloud resources for Jenkins management and testing agents, the Cycle Appliance furnishes pre-configured infrastructure tailored for running Cycle tests in your cloud environment. Its primary objective is to streamline the setup process, significantly reducing the time required for customers to establish the necessary testing infrastructure and enabling prompt initiation of Cycle tests. It's worth noting that Cycle provides optimal value when employed within a CI/CD mindset, and the Cycle Appliance is designed to deliver this seamlessly out of the box.

Leveraging Terraform, the Cycle Appliance automates the setup and configuration of cloud infrastructure. By utilizing Terraform as an infrastructure-as-code language, it provisions and configures resources within the selected cloud provider. Given the slight variations in offerings among cloud providers, our Terraform code is meticulously adapted to meet the specific requirements for executing Cycle tests. Our overarching goal is to ensure consistency in deployment across all supported cloud providers, thereby enhancing interoperability and user experience.


#### Key Features

- Utilizes [Terraform](https://www.terraform.io/), an open-source infrastructure-as-code language developed by HashiCorp.
- Automated deployment of required infrastructure for autonomous Cycle testing in Microsoft Azure or Amazon AWS cloud environments.
- Automated configuration of the [Jenkins](https://jenkins.io) environment to allow it to integrate with the public cloud provider for dynamically provisioning testing agents.
- Customized Terraform code tailored to each cloud provider's specifications, ensuring optimal resource utilization.


### What Cycle Appliance for AWS builds
***

The architecture diagram for the Cycle Appliance is below; these resources are built and configured with the Terraform code. This diagram covers building a new VPC, but all other parts are relevant to the existing vpc variant.

![Image](https://cycldocimgs.blob.core.windows.net/docimgs/aws-cyclapp.png)


### IAM Role
***

The Cycle Appliance can dynamically spin up and spin down testing agents from Instances. This is achieved by the use of an (AWS IAM Role)[https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html] that is created and assigned to the Jenkins Manager Instance. This IAM Role is enabled on the Instance and given the policy contained in the `/scripts/jenkins-mgr-policy.json` file that is used for deploying the Cycle Appliance.

#### Key Features
- This allows the Jenkins manager to dynamically build testing agents using the [EC2](https://plugins.jenkins.io/ec2/) Jenkins plugin. This is a plugin to enable you to leverage their AWS cloud for testing agents and only pay for them while they're online.

### Cloud Initialization script
***

The `cloud-init` script (`cloud-init-tf.yml`) is where all of the host-based configuration happens. We use this scripting language to configure the Jenkins Manager Instance after it has been deployed into the cloud. This `cloud-init` script gets generated by Terraform, encoded as `base64`, with all of the variables injected into it, and then passes it into the Instance as it's deployed via the `custom_data` property. This way, right after the cloud provider builds the Instance resource, it executes the `cloud-init` script and configures that Instance to its desired state.

#### Key Features
- Updates all packages to ensure the Ubuntu Instance is up to date with the latest packages.
- Configures Jenkins fully with a Jenkins Configuration as Code file (jenkins.yaml)
- Installs required Jenkins plugins.
- A LOT of variables are sent into this `cloud-init` from the Terraform deployment itself (see below). This is the beauty of Terraform, we can easily interpolate all of this data and put it in place in our script files to allow for a way better configuration-as-code. 

```
data "cloudinit_config" "server_config" {
  gzip          = true
  base64_encode = true
  part {
    content_type = "text/cloud-config"
    content = templatefile("${path.module}/scripts/cloud-init-tf.yml", {
      "jenkinsadmin"            = var.jenkinsadmin
      "jenkinspassword"         = var.jenkinspassword
      "agentadminusername"      = var.agentadminusername
      "agentadminpassword"      = var.agentadminpassword
      "jenkinsserverport"       = "http://${var.jenkins_ip}:8080/"
      "jenkinsserver"           = var.jenkins_ip
      "aws_region"              = var.region_name
      "aws_subnet_id"           = data.aws_subnet.sn.id
      "aws_windows_agent_sg_id" = aws_security_group.windows_agents_sg.id
      "ec2_private_key"         = tls_private_key.windows_agents.private_key_pem
      "ami_name"                = var.ami_name
      "ami_owner"               = var.ami_owner
      "name_prefix"             = var.resource_name_prefix
    })
  }
}
```

## Preparing for Deployment
***
Our Terraform code is broken out into two different deployment types: `newvpc` and `existingvpc`. `newvpc` is if you want a new VPC to be deployed and used for your Cycle Appliance. `existingvpc` is if you want to deploy the Cycle Appliance to an existing VPC and subnet. The necessary Terraform files for each of these two deployment types are under the appropriate subfolder. You should determine which deployment you want to use, and move work within that directory. 

### Installing Terraform
***

***Please note that if you are using Terraform Cloud, you should just simply clone the repository and store it on a source code tool that your Terraform Cloud is connected to. You can skip the below steps.***
If you are doing the Terraform deployment locally, the Terraform binary will need to be installed from wherever the deployment will take place. 

- Make sure you have Terraform installed locally, and that `Terraform` is in your `PATH`.
    - [HashiCorp Documentation on installing Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) - they explain it better than we can.

### Using a Remote State 
***

If you are not using Terraform Cloud and would like to store your Terraform State File in a remote location like an S3 bucket, please copy/paste the below code snippet and put it in the `terraform` block in the `awscaevpc.tf` file and update the values with your data. You can also find documentation for this [here](https://developer.hashicorp.com/terraform/language/settings/backends/s3).

```
  backend "s3" {
   bucket = "<bucketname>"
   key    = "<path/to/my/key>"
   region = "<region>"
  }
```

### How authentication and permissions are handled

Authentication methods will split into two basic camps for AWS and Terraform. Running your terraform from either **inside** or **outside** of an AWS resource.

Regardless of where you're running the code from, you'll need the appropriate or greater permissions assigned to either the user or role that you're using to authenticate with. As there are loads of different resources you can create with Terraform, you can take the open approach and allow Terraform to do any operation needed with the below policy snippet.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "*",
            "Resource": "*"
        }
    ]
}
```

If you're looking to simply grant all permissions necessary to deploy the Cycle Appliance as we've defined it in this code, you can use the below AWS policy snippet.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "backup:CreateBackupPlan",
                "backup:CreateBackupSelection",
                "backup:CreateBackupVault",
                "backup:DeleteBackupPlan",
                "backup:DeleteBackupSelection",
                "backup:DeleteBackupVault",
                "backup:DeleteBackupVaultAccessPolicy",
                "backup:DeleteBackupVaultNotifications",
                "backup:DeleteRecoveryPoint",
                "backup:DescribeBackupJob",
                "backup:DescribeBackupVault",
                "backup:DescribeProtectedResource",
                "backup:DescribeRecoveryPoint",
                "backup:DescribeRestoreJob",
                "backup:ExportBackupPlanTemplate",
                "backup:GetBackupPlan",
                "backup:GetBackupPlanFromJSON",
                "backup:GetBackupPlanFromTemplate",
                "backup:GetBackupSelection",
                "backup:GetBackupVaultAccessPolicy",
                "backup:GetBackupVaultNotifications",
                "backup:GetRecoveryPointRestoreMetadata",
                "backup:GetSupportedResourceTypes",
                "backup:ListBackupJobs",
                "backup:ListBackupPlans",
                "backup:ListBackupPlanTemplates",
                "backup:ListBackupPlanVersions",
                "backup:ListBackupSelections",
                "backup:ListBackupVaults",
                "backup:ListProtectedResources",
                "backup:ListRecoveryPointsByBackupVault",
                "backup:ListRecoveryPointsByResource",
                "backup:ListRestoreJobs",
                "backup:ListTags",
                "backup:PutBackupVaultAccessPolicy",
                "backup:PutBackupVaultNotifications",
                "backup:StartBackupJob",
                "backup:StartRestoreJob",
                "backup:StopBackupJob",
                "backup:TagResource",
                "backup:UntagResource",
                "backup:UpdateBackupPlan",
                "backup:UpdateRecoveryPointLifecycle",
                "backup:UpdateRegionSettings",
                "backup-storage:MountCapsule",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:CreateNetworkInterface",
                "ec2:CreateSecurityGroup",
                "ec2:CreateTags",
                "ec2:CreateSecurityGroup",
                "ec2:CreateTags",
                "ec2:DeleteKeyPair",
                "ec2:DeleteNetworkInterface",
                "ec2:DeleteSecurityGroup",
                "ec2:DeleteTags",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeImages",
                "ec2:DescribeInstanceAttribute",
                "ec2:DescribeInstanceCreditSpecifications",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeInstances",
                "ec2:DescribeKeyPairs",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeRegions",
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeTags",
                "ec2:DescribeVolumes",
                "ec2:DescribeVpcAttribute",
                "ec2:DescribeVpcs",
                "ec2:DetachNetworkInterface",
                "ec2:GetPasswordData",
                "ec2:ImportKeyPair",
                "ec2:RevokeSecurityGroupEgress",
                "ec2:RunInstances",
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:TerminateInstances",
                "iam:AddRoleToInstanceProfile",
                "iam:AttachRolePolicy",
                "iam:CreateInstanceProfile",
                "iam:CreatePolicy",
                "iam:CreateRole",
                "iam:DeleteInstanceProfile",
                "iam:DeletePolicy",
                "iam:DeleteRole",
                "iam:DetachRolePolicy",
                "iam:GetInstanceProfile",
                "iam:GetPolicy",
                "iam:GetPolicyVersion",
                "iam:GetRole",
                "iam:ListAttachedRolePolicies",
                "iam:ListInstanceProfilesForRole",
                "iam:ListPolicyVersions",
                "iam:ListRolePolicies",
                "iam:PassRole",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:TagPolicy",
                "iam:TagRole",
                "kms:CreateGrant",
                "kms:Decrypt",
                "kms:DescribeKey",
                "kms:GenerateDataKey",
                "kms:RetireGrant"
            ],
            "Resource": "*"
        }
    ]
}
```

If you're running the build from within AWS, we highly suggest assigning an IAM role with appropriate permissions to the EC2 Instance where Terraform runs. Terraform can automatically use the IAM Instance profile associated with the Instance to authenticate with AWS services.

#### Outside AWS method

You'll need to configure AWS CLI with your region, access key id, and secret access key before running your `terraform apply`. You can use the below commands with your data in the `<input>` fields for your region, access key id, and secret access key.

- `aws configure set region <input>`
- `aws configure set aws_access_key_id <input>`
- `aws configure set aws_secret_access_key <input>`

With the above configuration commands complete on your machine, Terraform will use this AWS CLI profile to authenticate for all of its actions.

Alternatively, you can just set the following environment variables with your data in the `<input>` fields for your region, access key id, and secret access key. You can find out how to set environment variables in shell [here](https://www.digitalocean.com/community/tutorials/how-to-read-and-set-environmental-and-shell-variables-on-linux) and powershell [here](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_environment_variables?view=powershell-7.4).

- `AWS_ACCESS_KEY_ID=<input>`
- `AWS_SECRET_ACCESS_KEY=<input>`
- `AWS_DEFAULT_REGION=<input>`

We strongly advise you not to put these secret values into any code. Defining these values in those files may work, but it is far from best practice.

#### Inside AWS method

This method is preferred as it is more secure, so we recommend it whenever the environment allows. It's a very simple process. You'll just assign whatever resource you're running Terraform from an IAM role and attach the above policy snippet to the role. You can learn more about assigning an IAM role to an AWS resource [here](https://aws.amazon.com/blogs/security/easily-replace-or-attach-an-iam-role-to-an-existing-ec2-Instance-by-using-the-ec2-console/).

### Terraform Files
***

- `*.main.tf`: This is the file that declares what infrastructure should be provisioned by Terraform. 
- `*.vars.tf`: This file declares your variables, variable types, and some have default values that we recommend.
- `*.tfvars`: This file defines your variables by assigning them a value using a simple key/value pair format.
- `*.output.tf`: This file tells Terraform what to output to the end-user after the deployment is complete.

### The Variables

The main variables file that you'll need to focus on is `*.tfvars`. We have it pre-loaded with all required variables that must be assigned a value. Other variables have a default value loaded into our `*.vars.tf` file but feel free to override those values in the `*.tfvars` file.

These variable names should be pretty self-explanatory and have descriptions that explain what they do in the `*.vars.tf` file and listed below. They are split into groups that explain how they are used.

#### Variables with no default values

The below variables are pre-loaded in the `*.tfvars` file and must be assigned a value

**Existing & New VPC**
- `jenkins_ip` - ***(string)*** IP address for the Jenkins manager _(Example: 10.0.0.10)_
- `vpc_cidr` - ***(string)*** The CIDR of the VPC the Jenkins manager and agents will be on _(Example: 10.0.0.0/16)_
- `region_name` - ***(string)*** The region you'll be deploying these resources to _(Example: us-east-1)_
- `jenkinsadmin` - ***(string)*** Default Jenkins Admin user _(Example: admin)_
- `jenkinspassword` - ***(string)*** Default Jenkins Admin password _(Example: Pa$$w0rd!)_
- `ami_name` - ***(string)*** The AMI name that you'd like to use for pulling latest for the Jenkins agents _(Example: cycleready-2024-02-27)_
- `ami_owner` - ***(string)*** The OwnerID value from AWS that is assigned to the AMI you'd like to use for Jenkins agents _(Example: 123456789012)_
- `env_tag` - ***(string)*** The environment this resource is deployed in _(Example: prod)_
- `owner_tag` - ***(string)*** The person who created the resource _(Example: john-doe)_

**Existing VPC only**
- `sn_cidr` - ***(string)*** The CIDR of SN the Jenkins manager and agents will be on _(Example: 10.0.1.0/24)_

**New VPC only**
- `az_name` - ***(string)*** Name of the availability zone where resources will be deployed. _(Example: us-east-1a)_
- `private_sn_cidr` - ***(string)*** CIDR block for the private subnet behind the NAT gateway. _(Example: 10.0.1.0/24)_
- `nat_sn_cidr` - ***(string)*** CIDR block for the subnet where the NAT gateway will reside. _(Example: 10.0.2.0/24)_
- `network_prefix` - ***(string)*** Prefix used for naming network resources. _(Example: cyclenet)_


#### Variables with default values
The below variables are not pre-loaded in the `*.tfvars` as they have default values in the `*.vars.tf` file, but feel free to override any of these default values by pulling the key/value pairs from the snippet below these descriptions.

**Existing & new VPC**
- `allow_cidrs` - ***(list)*** Optional CIDRs you want to allow to access the Jenkins manager in your security group _(Example: ["192.168.1.0/24", "10.1.1.0/24"])_
- `resource_name_prefix` - ***(string)*** A prefix for the name of all resources deployed with this code _(Example: cycleappliance)_
- `mgr_ssh_key_name` - ***(string)*** Name of the manager SSH keypair file(s) within the /keys/ directory _(Example: ca-mgr)_
- `agent_ssh_key_name` - ***(string)*** Name of the agent SSH keypair file(s) within the /keys/ directory _(Example: ca-agent)_
- `Instance_type` - ***(string)*** Instance type used for the EC2 Instance for the Jenkins manager _(Example: t2.small)_
- `volume_size` - ***(number)*** Size in GB for the Jenkins manager OS disk _(Example: 30)_
- `backup_schedule` - ***(string)*** The schedule in cron format to use for backing up the Jenkins manager _(Example: cron(0 5 ? * MON-FRI *))_
- `backup_retention_days` - ***(number)*** The amount of days to retain of Jenkins manager backups _(Example: 7)_
- `agentadminusername` - ***(string)*** The agent Admin username needed to connect to the agent with Jenkins (We've defaulted to using the default username defined in our Packer Template for CycleReady Images for your convenience) _(Example: administrator)_
- `agentadminpassword` - ***(string)*** The agent Admin password needed to connect to the agent with Jenkins (We've defaulted to using the default password defined in our Packer Template for CycleReady Images for your convenience) _(Example: 68q79h4W#mN4k87P#!JQ)_

```
allow_cidrs             = "<input>"
resource_name_prefix    = "<input>"
Instance_type           = "<input>"
volume_size             = "<input>"
backup_schedule         = "<input>"
backup_retentation_days = <input>
agentadminusername      = "<input>"
agentadminpassword      = "<input>"
ami_name                = "<input>"
```

### Generating SSH keypair
***

Our code will generate your two SSH keypairs using `tls_private_key` resource blocks in the `*.main.tf`. One of them will be for accessing the Jenkins manager and the other will be used for Windows agents that the Jenkins manager spins up. It will store these keypairs in the `/keys` directory using the default values for vars `var.mgr_ssh_key_name` and `var.agent_ssh_key_name` in the `*.vars.tf` file. You will need to store these keys in a safe place once generated on `terraform apply` and use them for SSH access.

### Networking Considerations
***

Due to the complexities and requirements of our customers, we built our Cycle Appliance to be deployed in two different styles: onto an entirely new VPC, or to join an existing VPC. Within the root folder of the repository, you will see a Cycle Appliance folder for both of these configuration types: `cycle-appliance-newvpc` and `cycle-appliance-existingvpc`.

The Jenkins agent Security Group this builds is set to allow all traffic from the private CIDR. If you'd like to scope this down using the Principle of Least Privilege, the necessary ports for Jenkins manager to agent connection are listed below. 

WinRM: TCP 5985
WinRM (HTTPS): TCP 5986
SMB: 445

Once again, we created a default blank var named `var.allow_cidrs`. Please add any additional CIDRs you'd like to allow access to the Jenkins manager as a list value to this var to conveniently allow resources on those CIDRs to immediately have access once the Jenkins manager is deployed.

#### Deploying to a new VPC

With a new network deployment, we've included every resource you'd need to create for the Jenkins manager to have internet access with no public internet ingress. You will need to ensure that you assign appropriate values for `vpc_cidr`, `private_sn_cidr`, and `nat_sn_cidr`. Both the `sn_cidr` values should be within the `vpc_cidr` scope.

The Jenkins manager and all of the testing agents will be deployed into `private_sn` and the `nat_sn` will serve as a public space with the NAT gateway and internet gateway to ensure public egress with no public ingress. This means that you'll need to either peer this VPC to another VPC that you have private access to or set up a VPN to AWS with access to this new VPC so you can have private access. If you prefer to set Jenkins manager up for public access, you can modify our code slightly to assign it a public IP address and put it on the `nat_sn` subnet, instead of the `private_sn` subnet. More info on this can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html#concepts-public-addresses)

If you're deploying an AWS network for the first time, please refer to the link [here](https://docs.aws.amazon.com/appstream2/latest/developerguide/managing-network-internet-NAT-gateway.html) that will go over the basics. Our code will build everything you need to get going, but reading through this will give you a better idea of what it's doing and how you should architect it. Here is a diagram of the most basic design to serve privately-connected EC2 instances.

![image](https://cycldocimgs.blob.core.windows.net/docimgs/aws-basic-nat-net.png)

#### Deploying to an existing VPC

If deploying to an existing VPC, you will need to take into consideration regional requirements in AWS. If your existing VPC is on the US-EAST-1 region, you should also deploy your Cycle Appliance to the US-EAST-1 region. This is a good option for customers who have an existing network within AWS that already has connectivity set up to their WMS environments, so we wanted to allow customers to just put their Cycle Appliance onto that existing network architecture. 

## Deploying the Cycle Appliance with Terraform
***

### Local Deployment
1. Change directories into the Cycle Appliance project folder; `./cycle-appliance-newvpc` or `./cycle-appliance-existingvpc`.
2. Run `terraform init` to initialize the project. This will download the providers and get `terraform` ready to run `plan` and `apply` commands.
3. Run `terraform validate` to make sure there are no syntaxual issues with any of the `*.tf` files.
4. Run `terraform plan` to generate a tentative plan of what will be deployed. Since we do not store passwords in the `*.vars.tf` file, it will prompt you to set the `agentadminpassword` and `jenkinspassword` variables at runtime unless you defined them in the `*.tfvars` file.
    - ![Image](https://cycldocimgs.blob.core.windows.net/docimgs/plan.png)
    - `agentadminpassword` is the password that will be set for the local admin accounts for the Windows agent Instances; it needs to be `12` characters long or more to meet AWS requirements.
    - `jenkinspassword` is the password that will be set for the Jenkins administrator account that we create. This has no length requirements.
5. Your `terraform plan` will output what will be built; so you can review this and make sure everything looks good. If it does, you can proceed.
6. Finally, you can run `terraform apply` to deploy the resources to AWS. Terraform will show you the output of everything it is building and tell you when it's completed the deployment of each resource. This process will take about 3 minutes to complete.
    - ![Image](https://cycldocimgs.blob.core.windows.net/docimgs/aws-apply.png)
7. Once the `terraform apply` is finished, the initialization script will still be running on the Jenkins manager to do a lot of the heavy lifting. You should allow `5 minutes` before connecting this server to try and do everything.
    - If you deployed the `./cycle-appliance-newvpc` configuration, you will need to peer that private VPC with a network you can access before you can SSH into the box.
8. You will be able to access the Jenkins Manager using the private IP, on port `8080` - or just by using the `jenkins_manager` output value.
    - You'll be able to log in with the values for `var.jenkinsadmin` and `var.jenkinspassword`. 
    - OR if you want to access the Jenkins Manager Instance with SSH; you can use the private key you created with the `var.jenkinsadmin` username. The automation will associate the public key that you placed in the `/keys` folder with that username.
9. Our automation configures the Jenkins Manager so that it's about 95% of the way to being configured to start testing with Cycle. The last step is altering the image that is used for agent creation. By default, we set this to use the standard Windows Server 2019 DataCenter from the AWS marketplace. This is great to test that Jenkins is able to talk back to AWS, and provision / deprovision testing agents. You will want to create a golden image that has Cycle and other necessary tools on it, and then utilize that image within Jenkins. Thankfully, we've also built automation to help you do this using Packer. That documentation is located [here](https://dev.azure.com/cyclelabs/cycle-codetemplates/_git/packer?path=/aws/cycleready) 
    - Once you create this image, you can configure it to be used in the **Cloud Config** area of **Manage Jenkins**.
    
### Terraform Cloud Deployment


## Help, Questions, and Feedback
If you need assistance, have questions, or want to provide feedback - we're here for you! You can reach our Cloud Engineering team at: [cloudengineering@cyclelabs.io](mailto:cloudengineering@cyclelabs.io?subjectCycle%20Appliance@Help)

## Change Log 
- 06/21/2024 - We added the `os_type` variable to the Cycle Appliance for AWS. This variable can be set to either `ubuntu` or `redhat` and depending on the value, it will properly deploy the latest version of the selected operating system (Ubuntu 22.04, and RHEL 9.4) and also handle all of the post configuration via the respective `cloud-init` script. For Ubuntu, it will use the `cloud-init-tf.yml`, and for Red Hat, it will use the `rhel-cloud-init-tf.yml`.