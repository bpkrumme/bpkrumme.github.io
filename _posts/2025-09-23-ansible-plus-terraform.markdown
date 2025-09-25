---
layout: single
title:  "Simple Cloud Provisioning and Automation with Terraform and Ansible - Part 1: Terraform"
date:   2025-09-23
header:
  og_image: /assets/images/bdt_square.jpeg
---

## There's more than one best tool...

Terraform and Ansible working together is a subject I get asked about fairly regularly.  Most of the time, the way the conversation starts is an objection masked as a question, and it is usually an engineer who has an affinity for Terraform as their Infrastructure-as-code tool of choice.  They really want to know why they should consider Ansible for part of their provisioning process, instead of sticking with what they know.  I don't blame them, either, as Terraform excels at provisioning infrastructure, especially in hyperscalers such as AWS, Azure, and GCP.

Luckily, this conversation is changing due to the acquisition of HashiCorp by IBM.  The focus is shifting to using the two tools together, and leveraging them for their strengths.  Let's take a look Terraform and its strengths, as well as those it shares with Ansible.

### Terraform Strengths

* Infrastructure as Code (IaC): Terraform allows you to define and provision infrastructure resources via declarative configuration.
* Multi-cloud Support: It supports multiple cloud providers, including AWS, Azure, Google Cloud, and more.
* Stateful Operation: Terraform can manage the state of infrastructure, keeping track of what resources have been created and their current state.
* Parallel Execution: Terraform can execute multiple resources in parallel, reducing the time it takes to provision infrastructure.

### Shared Strengths with Ansible

* Community and Ecosystem: Both Terraform and Ansible have large and active communities, with many third-party modules and tools available.
* Integrations: Both Ansible and Terraform can integrate with other tools such as with CI/CD pipelines.
* Modularity and Reusability: Terraform and Ansible both allow you to create modular and reusable code, making it easier to manage complex infrastructure deployments and their configurations.

The tools complement each other nicely, with Terraform's primary use case as an Infrastructure-as-Code state machine and Ansible's use case as the orchestrator of that infrastructure's configuration.  It's also convenient that both Ansible and Terraform share some strengths which make them powerful weapons for the IT lifecycle management arsenal. Projects for both can be stored in a version control system such as git, allowing for change tracking, collaboration amongst members of a team, and branching/merging/review of new features and bug fixes.

## Automating Provisioning with Terraform

Infrastructure provisioning, especially in the public cloud is where Terraform really shines.  I'll show you the basics of Terraform and how to provision a basic environment.

### Prerequisites

Before I can actually start defining some infrastructure as code, I need to have some tools installed and configurations set.  Here's a quick list:

Necessary Tools:

* Terraform CLI
* Cloud CLI
* Cloud CLI configured for the selected hyperscaler
* A text editor

Optional Tools and Add-Ons:

* An IDE (Integrated Development Environment) such as Visual Studio Code
* Git or other source control
* Terraform Extension for Visual Studio Code
* Ansible Development Tools and Ansible Lightspeed
* Terraform Cloud Development Kit or MCP Server

It's extremely helpful to use a development environment you are familiar with.  I am very experienced working in a Linux shell and could do all of my development using vi, however many tools available today have features like automatic syntax highlighting, inline debugging, and git integration making the experience a bit more seamless.

My IDE of choice is Visual Studio Code with several useful extensions installed including the HashiCorp Terraform, Remote Development, GitLens, and Code Spell Checker extensions to name a few.

### A basic Terraform project for AWS

A simple Terraform project consists of several files which define the Terraform environment and the infrastructure resources we want to deploy.  I'm most familiar with AWS, so let's deploy some resources there.

Here's what the project directory looks like:

```text
└── terraform_basic
    ├── main.tf
    ├── outputs.tf
    ├── terraform.tf
    └── variables.tf
```

#### terraform.tf

Let's start with the `terraform.tf` file, which defines the required version of Terraform, the binary plugins called providers which call the cloud provider's API to manage resources, and other configurations such as where to store Terraform state.  Terraform uses its own syntax, which is hierarchical in nature.  Let's take a look at my configuration.

```json
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.14"
    }
  }
  required_version = ">= 1.12"
  backend "s3" {
    bucket = "rhbtfstate"
    key    = "terraform/basic/env/state"
    region = "us-east-2"
  }
}
```

The beginning of the `terraform.tf` configuration starts with the top-level `terraform` block.  Inside that block, I have 3 configuration blocks which define my desired Terraform configuration.  These can be defined in any order.

##### required_providers

The first block I have defined is the `required_providers` block which defines the provider plugins needed to create and manage the resources in my configuration.  Because I want to deploy to AWS, I've defined `aws` as a required provider.  That definition has two arguments, source and version.

```json
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.14"
    }
  }
```

The source argument is the global source address of the provider I want to use which consists of 3 parts delimited by slashes:

```text
[<HOSTNAME>/]<NAMESPACE>/<TYPE>
```

You'll notice that my configuration does not include a hostname.  This is because I want to use the public Terraform Registry, which is the default.  If I wanted to use a private registry, I could define the hostname here.

The version argument is the constraint I have chosen for the version of the hashicorp/aws provider, set to greater than or equal to version 6.14.

##### required_version

The second configuration is the `required_version` which specifies the version of the terraform CLI which is allowed to run the configuration.

```json
  required_version = ">= 1.12"
```

##### backend

The final block in my `terraform.tf` configuration file is the backend I have defined for storing state files.  If you don't define where your state is stored, it gets written locally, which is ok for local development or small isolated projects, but because I want to be able to use the Terraform state for other purposes, like my Ansible inventory, storing it in Amazon S3 makes sense.  Keep in mind that you can only define one backend.

For the s3 backend type, there are several configuration parameters including `region` which is the AWS region for your s3 bucket, `bucket` which is the bucket name, and `key` which is the path to the state file inside the S3 bucket.  All of these parameters are required for S3 stage storage.

```json
  backend "s3" {
    bucket = "rhbtfstate"
    key    = "terraform/basic/env/state"
    region = "us-east-2"
  }
```

There are other specific backend types you can use should you not want to use S3, take a look here for more detail:  [Terraform State Backends](developer.hashicorp.com/terraform/language/backend)

#### main.tf

The next important file in our Terraform project is the `main.tf` file, which defines the infrastructure objects and resources we want to create using Terraform.  My basic project includes 3 different code block types.

##### provider

The `provider` block defines the configuration for the provider plugin(s) which were already defined in `terraform.tf`.  You might have multiple `provider` blocks in your configuration.  I only have one because every resource is created in AWS.  Also, in my case, I'm only setting one parameter for the AWS provider, which is the EC2 Region I want to deploy resources in.

```json
provider "aws" {
  region = var.ec2_region
}
```

You might notice that the value for the region is set to `var.ec2_region`.  This is because I want my configuration to be flexible, so I use variables for much of the configuration.  We'll take a look at variables in the next section.

##### data

The `data` block is how you fetch data about a resource in your chosen provider without doing any provisioning.  This is especially useful for referencing attributes from one object to dynamically configure other objects.  Here, I am fetching data about the Amazon Machine Image I want to use when deploying EC2 Instances.  I'm looking for the latest AMIs owned by Red Hat, using their account ID found in their AMI details, and I'm filtering by AMI name to include only RHEL-10, as well as filtering by architecture for only x86_64.

```json
data "aws_ami" "rhelami" {
  most_recent = true
  owners      = ["309956199498"] # Red Hat

  filter {
    name   = "name"
    values = ["RHEL-10*-x86_64-*"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}
```

##### resource

There will almost certainly be multiple `resource` blocks in your configuration.  In my configuration, I have a total of 9 resource blocks which define all of the necessary infrastructure components in AWS from a VPC all the way to the EC2 instances I want to deploy.  The resource block defines the type of resource you want to create, update, or delete and the parameters for that resource.  The resource types are defined by the provider(s) you have configured in your project and their parameters vary.  You will need to refer to the provider documentation for each of the resources you want to deploy.

Here is an example from my configuration, which defines the EC2 Instances I want to deploy.  The resource type is `"aws_instance"`, I've named this resource `"tfservers"`, and I have multiple parameters set, including the AMI ID which was found using the data block above, the number of instances, the instance type, networking and security parameters, and tags.

```json
resource "aws_instance" "tfservers" {
  ami                         = data.aws_ami.rhelami.id
  count                       = var.number_of_instances
  instance_type               = var.instance_type
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.bdtsg.id]
  key_name                    = var.instance_key
  tags = {
    Name        = "${var.instance_name}${count.index}.${var.instance_domain_name}"
    Environment = var.instance_environment
    Region      = var.ec2_region
  }
}
```

You can see that I'm also using variables here, and there are multiple references to other resources which are being created earlier in the configuration.  Here is my complete `main.tf` file:

```json
provider "aws" {
  region = var.ec2_region
}

resource "aws_vpc" "bdtvpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = false
  tags = {
    Name = "Brad-Does-Tech-VPC"
  }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.bdtvpc.id
  cidr_block = "10.0.1.0/24"
  tags = {
    Name = "Brad-Does-Tech-Subnet-Public"
  }
}

resource "aws_route_table" "bdtrt" {
  vpc_id = aws_vpc.bdtvpc.id
  tags = {
    Name = "Brad-Does-Tech-Route-Table"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.bdtrt.id
}

resource "aws_internet_gateway" "bdtigw" {
  vpc_id = aws_vpc.bdtvpc.id
  tags = {
    Name = "Brad-Does-Tech-Internet-Gateway"
  }
}

resource "aws_route" "internet-route" {
  destination_cidr_block = "0.0.0.0/0"
  route_table_id         = aws_route_table.bdtrt.id
  gateway_id             = aws_internet_gateway.bdtigw.id
}

resource "aws_security_group" "bdtsg" {
  name        = "Brad-Does-Tech-SG"
  description = "Allow inbound traffic"
  tags = {
    Name = "Brad-Does-Tech-Inbound-SG"
  }
  vpc_id = aws_vpc.bdtvpc.id
  ingress {
    description = "SSH"
    from_port   = "22"
    to_port     = "22"
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    description = "HTTPS"
    from_port   = "443"
    to_port     = "443"
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    cidr_blocks = ["0.0.0.0/0"]
    from_port   = "0"
    protocol    = "-1"
    to_port     = "0"
  }
}

data "aws_ami" "rhelami" {
  most_recent = true
  owners      = ["309956199498"]

  filter {
    name   = "name"
    values = ["RHEL-10*-x86_64-*"]
  }
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }
}

resource "aws_instance" "tfservers" {
  ami                         = data.aws_ami.rhelami.id
  count                       = var.number_of_instances
  instance_type               = var.instance_type
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.bdtsg.id]
  key_name                    = var.instance_key
  tags = {
    Name        = "${var.instance_name}${count.index}.${var.instance_domain_name}"
    Environment = var.instance_environment
    Region      = var.ec2_region
  }
}

resource "aws_route53_record" "server_dns" {
  count   = var.number_of_instances
  zone_id = "${var.hosted_zone_id}"
  name    = "${var.instance_name}${count.index}.${var.instance_domain_name}"
  type    = "A"
  ttl     = 300
  records = [aws_instance.tfservers[count.index].public_ip]
}
```

Because this configuration relies heavily on variables, let's take a look at variables now.

#### variables.tf

The `variables.tf` file declares variable data we want to use in our project.  There will be one `variable` block for each piece of data we want to use.  The cool part about variables is they are dynamic in nature, so we can override the default values if we want, or we can declare variables without a default, and be prompted for the data when we run our Terraform plan.

```json
variable "instance_key" {
  description = "The key used for the EC2 instance."
  type        = string
  default     = "awskeypair"
}

variable "number_of_instances" {
  description = "The number of EC2 instances."
  type        = number
  default     = 4
}
```

Each of the variable examples here are used for configuring the EC2 instance resources shown above and both have default values.  I do want to point out that there are multiple types of variables.  You'll notice that the `"instance_key"` variable is a string while the `"number_of_instances"` variable is a number.  Declaring the correct variable type is important to ensure your plan runs without errors.  If `"number_of_instances"` was declared as a string instead of a number, the Terraform plan would fail.

Here is my complete `variables.tf` file:

```json
variable "ec2_region" {
  description = "Region for our EC2 instances."
  type        = string
  default     = "us-east-2"
}
variable "instance_name" {
  description = "Value of the EC2 instance's Name tag."
  type        = string
  default     = "tfserver"
}

variable "instance_type" {
  description = "The EC2 instance's type."
  type        = string
  default     = "t3.micro"
}

variable "instance_key" {
  description = "The key used for the EC2 instance."
  type        = string
  default     = "awskeypair"
}

variable "number_of_instances" {
  description = "The number of EC2 instances."
  type        = number
  default     = 4
}

variable "instance_domain_name" {
  description = "The domain name for EC2 instances."
  type        = string
  default     = "braddoestech.com"
}

variable "hosted_zone_id" {
  description = "The hosted zone id for Route 53"
  type        = string
  default     = "<<REDACTED>>"
}
variable "instance_environment" {
  description = "The environment for grouping EC2 instances."
  type        = string
  default     = "Development"
}
```

#### outputs.tf

The last piece, which is optional, is the `outputs.tf` file.  This file defines any data you would like expose about your infrastructure which is deployed or updated.  It is especially helpful when you are automating against the infrastructure later with Ansible.

In my case, I only want one extra piece of data exposed, which is the DNS names of each EC2 instance I've created.  I don't need a special format, so I'm happy to expose this as a simple array.  Here's how I do that:

```json
output "instance_hostname" {
  description = "DNS name of the EC2 instance."
  value       = aws_route53_record.server_dns.*.name
}
```

### Running Terraform

Now that you have your configuration, you can initialize and apply the configuration.  Here's a quick rundown of the steps:

1. Ensure your Cloud Provider CLI is installed and configured

2. Ensure your Cloud Provider Credentials are correct and available in your environment.  For AWS, you can set the `AWS_SECRET_ACCESS_KEY` and `AWS_ACCESS_KEY_ID` environment variables to accomplish this

3. Run `terraform init` in your project directory to initialize the terraform environment and ensure the provider(s) you have configured are available in your environment

    ```shell
    $ terraform init
    Initializing the backend...

    Successfully configured the backend "s3"! Terraform will automatically
    use this backend unless the backend configuration changes.
    Initializing provider plugins...
    - Finding hashicorp/aws versions matching "~> 6.14"...
    - Installing hashicorp/aws v6.14.1...
    - Installed hashicorp/aws v6.14.1 (signed by HashiCorp)
    Terraform has created a lock file .terraform.lock.hcl to record the provider
    selections it made above. Include this file in your version control repository
    so that Terraform can guarantee to make the same selections by default when
    you run "terraform init" in the future.

    Terraform has been successfully initialized!
    ...
    ```

4. Run `terraform apply` in your project directory to apply the configuration and create or update the resources.  Be sure to type "yes" when prompted so your plan actually runs.  Any other response here will cancel the run.

    ```shell
    $ terraform apply
    data.aws_ami.rhelami: Reading...
    data.aws_ami.rhelami: Read complete after 0s [id=ami-0f70b01eb0d5c5caa]

    Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
      + create

    Terraform will perform the following actions:

    ...(truncated)

    Do you want to perform these actions?
      Terraform will perform the actions described above.
      Only 'yes' will be accepted to approve.

      Enter a value: yes

    aws_vpc.bdtvpc: Creating...
    aws_vpc.bdtvpc: Creation complete after 2s [id=vpc-00ad167c7217d08e2]
    aws_route_table.bdtrt: Creating...

    ...(truncated)

    Apply complete! Resources: 15 added, 0 changed, 0 destroyed.

    Outputs:

    instance_hostname = [
      "tfserver0.braddoestech.com",
      "tfserver1.braddoestech.com",
      "tfserver2.braddoestech.com",
      "tfserver3.braddoestech.com",
    ]

    ```

5. Should you want to update the configuration, simply modify the confiuration and run `terraform apply` again.

6. In the event you want to completely remove the infrastructure, you can run `terraform destroy` to delete all of the objects which were created previously.

### Terraform Documentation and Resources

I understand that my examples here may not be 100% complete, and I don't want you fumbling through your first attempts at running a Terraform project.  Here are some resources to help.

#### Tutorials

So you aren't completely lost, I highly recommend taking a look at the Terraform Tutorials which are available for free on the HashiCorp developer site.  You can find them here:  [Terraform Tutorials](https://developer.hashicorp.com/terraform/tutorials)

#### Documentation

The Terraform documentation is quite good and is excellent at explaining the multitude of configuration parameters available.  Check it out here:  [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)

#### My Example Project

If you would like to use my basic project as a reference you can find the full configuration here:  [Project source on GitHub](https://github.com/bpkrumme/terraform_basic)

### Up Next

Be on the lookout for part 2, where I will use Ansible to apply Terraform, then create a workflow in Automation Controller to apply the Terraform and also configure the servers.