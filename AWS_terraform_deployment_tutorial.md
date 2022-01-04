# edX AWS Deployment Tutorial using Terraform

This tutorial will walk you through deploying edX to Amazon Web Services.

We will be using Terraform for setting up the AWS resources and Ansible to automate deployment.

## Prerequisites:

- **IAM Role Sign On From OpenCraft Account**: If this is a new AWS account for a new client, you will 
want to set up access to it from the main AWS OpenCraft account as this allows all OpenCraft 
contractors to access the account. Go to roles (or ask the Client, if they are setting up the account)
in the new account and click "Create role". Fill it out like this:

  - Select type of trusted entity: Another AWS account
  - Account ID: The AWS account ID
  - Require external ID (Best practice when a third party will assume this role): False
  - Require MFA: True
  - Attach permissions policies: AdministratorAccess
  - Set permissions boundary: Create role without a permissions boundary
  - Role name: OpenCraft_Admin_Access
  
- **A [MongoDB Atlas Account](https://www.mongodb.com/cloud)**: There is a shared OpenCraft Account that 
can be used when setting up client instances. You can find the credentials for that in the vault 
[here](https://vault.opencraft.com:8200/ui/vault/secrets/secret/show/core/OpenCraft%20Tools%20:%20Resources%20:%20MongoDB%20Atlas).

- **AWS Region**: The client should decide on which region they want to host their resources.
- **Domain/SubDomain Delegation**: The client should decide what's the domain/subdomain that will point
to the edX instance (the root domain, where the LMS will live). After the domain is decided, they should
agree to delegate it to us, for which we'll need to provide some Nameservers (got from Route53, check
this step:)
- **Client Private Repository**: Create a new Subgroup [here](https://gitlab.com/opencraft/client) and
a repository called `configuration-secure` inside, this is where all the terraform and ansible scripts
and configuration for this client will live.
- **Two AWS Key Pairs**: [Create/Import 2 AWS Key Pairs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#how-to-generate-your-own-key-and-import-it-to-aws)
one will be used for access to the Director Instance, and the second one for the edX one.
- **Terraform**: [Install terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

## Terraform - Setting up AWS Resources

We'll be reusing the [`terraform-scripts`](https://github.com/open-craft/terraform-scripts) modules
along this tutorial, so you can also go there and check the modules to understand what each of the
modules do.

Inside the `configuration-secure` repository, create a `terraform` folder. We are going to create
two separate Terraform projects, one that it only needs to be setup once, and another that will
have the resources that the edX instance uses on a daily basis (`prod` for example):

    configuration-secure/
        terraform/
            setup/
            prod/

Go inside the `setup` folder and create a `main.tf` file with the following content:
    
    provider "aws" {
      region = <AWS Region>
      ...
    }

    module "route53" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/route53?ref=<LATEST RELEASE>"
    
      customer_domain = <CUSTOMER DOMAIN>
    }

    module "email" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/email?ref=<LATEST RELEASE>"
    
      customer_domain = <CUSTOMER DOMAIN>
      custom_emails_to_verify = <LIST OF EMAILS>
      internal_emails = <LIST OF INTERNAL EMAILS>
      route53_id = module.route53.route53_id
    
      customer_name = <CUSTOMER NAME>
      environment = <CUSTOMER ENVIRONMENT>
    }
    
    output "route53_name_servers" {
      value = module.route53.route53_DNS
    }

For more explanation on the input/output variables of each module, please check the README inside
[each module](https://github.com/open-craft/terraform-scripts/tree/main/modules).

- Customer Domain: The root domain to be configured
- List of Emails: Put here testing emails, that will receive emails while SES is in a sandbox:
      
      ["myname@opencraft.com"]
      
- List of Internal Emails: Emails that will be used by the edX instance, usually this should be:

      ["no-reply@<customer_domain>"]
      
- Customer Environment: Set it to `prod`

After configuring that first terraform project, run:

    terraform init
    terraform apply 
    
*Note: It's recommended at this point to configure a [terraform state backend](https://www.terraform.io/docs/backends/types/s3.html), 
so the state is not only stored in your computer (but S3 for example, or committing it to the repository)*

Terraform will output the Route53 Nameservers you'll need to share with the Customer for the domain name delegation.

---

Now go inside the `prod` folder and create a `main.tf` and a `director.pem` file that should contain
the director SSH private key you created previously and imported as Key Pair, your file structure so 
far should resemble this:

    configuration-secure/
        terraform/
            setup/
                main.tf
                ...
            prod/
                main.tf
                director.pem
                ...
                
Inside the `prod/main.tf` file, copy this (changing the respective values):

    provider "aws" {
      region = "<AWS Region>"
      ...
    }
    
    module "director" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/services/director?ref=<LATEST RELEASE>"
      director_key_pair_name = <AWS DIRECTOR KEY PAIR NAME>
      image_id = <AWS DIRECTOR IMAGE ID>
      instance_type = <AWS DIRECTOR INSTANCE TYPE>
    }
    
    module "s3" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/s3?ref=<LATEST RELEASE>"
      customer_name = <CUSTOMER NAME>
      environment = <CUSTOMER ENVIRONMENT>
    }
    
    module "openedx" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/services/openedx?ref=<LATEST RELEASE>"    
      customer_name = <CUSTOMER NAME>
      director_security_group_id = module.director.director_security_group_id
      environment = <CUSTOMER ENVIRONMENT>
      customer_domain = <CUSTOMER DOMAIN>
    }
    
    module "sql" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/services/sql?ref=<LATEST RELEASE>"
    
      customer_name = <CUSTOMER NAME>
      environment = <CUSTOMER ENVIRONMENT>
    
      allocated_storage = <SQL ALLOCATED STORAGE>
      database_root_username = <SQL ROOT USERNAME>
      database_root_password = <SQL ROOT PASSWORD>
    
      edxapp_security_group_id = module.openedx.edxapp_security_group_id
    
      instance_class = <SQL INSTANCE CLASS>
    }
    
    module "elasticsearch" {
      source = "git@github.com:open-craft/terraform-scripts.git//modules/services/elasticsearch?ref=<LATEST RELEASE>"
      customer_name = <CUSTOMER NAME>
      edxapp_security_group_id = module.openedx.edxapp_security_group_id
      environment = <CUSTOMER ENVIRONMENT>
    }
    
    resource aws_instance edxapp {
      ami = <AWS EDX IMAGE ID>
      instance_type = <AWS EDX INSTANCE TYPE>
      vpc_security_group_ids = [module.openedx.edxapp_security_group_id]
    
      root_block_device {
        volume_size = 50        # setting only 50 GB for the edxapp disk space
        volume_type = "gp2"
      }
    
      key_name = <AWS EDX KEY PAIR NAME>
    
      tags = {
        Name = "edxapp-1"  # keep adding resources and change the name
      }
    }
    
    // adding the edx instance into the load balancer
    resource aws_lb_target_group_attachment edxapp {
      target_group_arn = module.openedx.edxapp_lb_target_group_arn
      target_id = aws_instance.edxapp.id
      port = 80
    
      lifecycle {
        create_before_destroy = true
      }
    }
    
    // adding the edx instance into the main domain load balancer
    resource aws_lb_target_group_attachment "edxapp_main_domain" {
      target_group_arn = module.openedx.edxapp_main_domain_lb_target_group_arn
      target_id = aws_instance.edxapp.id
      port = 80
    }
    
    // OUTPUTS

    output "director_public_ip" {
      value = module.director.director_instance_public_ip
    }
    
    output "rds_host_name" {
      value = module.sql.mysql_host_name
    }
    
    output "elasticsearch_endpoint" {
      value = module.elasticsearch.elasticsearch
    }
    
    output "openedx_s3_storage_bucket_name" {
      value = module.s3.s3_storage_bucket_name
    }
    
    output "openedx_s3_storage_access_key" {
      value = module.s3.s3_storage_user_access_key
    }
    
    output "openedx_s3_storage_access_secret" {
      value = module.s3.s3_storage_user_secret_key
    }
    
    output "openedx_s3_tracking_logs_bucket_name" {
      value = module.s3.s3_tracking_logs_bucket_name
    }
    
    output "openedx_s3_tracking_logs_access_key" {
      value = module.s3.s3_tracking_logs_user_access_key
    }
    
    output "openedx_s3_tracking_logs_access_secret" {
      value = module.s3.s3_tracking_logs_user_secret_key
    }

- **AWS Director Key Pair Name**: Name of the Key Pair you previously created through AWS console that
will be used to SSH into the director instance.
- **AWS Director Instance Type**: Choose the smallest instance
- **SQL Allocated Storage**: Normally we set `5` by default for the allocated SQL storage
- **SQL Root Username**: Normally we set it to `opencraft`
- **SQL Root Password**: Secured Root Password, generate it and store, will be used later
- **SQL Instance Class**: Something similar or close to `db.m3.medium`
- **AWS edX Image ID**: Get an Image ID from AWS for Ubuntu 16.04
- **AWS edX Instance Type**: Something similar or close to `t2.large`
- **AWS edX Key Pair Name**: Name of the Key Pair you previously created through AWS console that will
be used for SSH into the edX instance from the director instance.

Run:

    terraform init
    terraform apply
    
Terraform should output information that will be used later to populate your `vars.yml` file. For
more information about those variables, please check the READMEs of each module in the 
[terraform-scripts repository](https://github.com/open-craft/terraform-scripts/tree/main/modules).

After this you should have every resource necessary for a basic edX instance, follow 
[this guide](https://gitlab.com/opencraft/documentation/public/-/blob/master/tutorials/howtos/aws/AWS_ELB_deployment_tutorial.md#mongodb-atlas-database) 
(from the MongoDB Atlas Database section onwards) to continue with the edX installation.
