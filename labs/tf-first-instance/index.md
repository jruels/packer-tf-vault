# Terraform Lab 1

# Overview
This lab walks through setting up Terraform and creating your first resources. 

## Install Terraform 
Create and enter the working directory:
```sh
mkdir -p $HOME/$(date +%Y%m%d)/terraform
cd $_
```
The AWS CloudShell does not have Terraform installed. To install the latest version run the following: 
```sh
TER_VER=`curl -s https://api.github.com/repos/hashicorp/terraform/releases/latest | grep tag_name | cut -d: -f2 | tr -d \"\,\v | awk '{$1=$1};1'`
wget https://releases.hashicorp.com/terraform/${TER_VER}/terraform_${TER_VER}_linux_amd64.zip
unzip terraform_${TER_VER}_linux_amd64.zip && sudo mv terraform /usr/local/bin/
```

Confirm installation was successful
```sh
terraform version 
```

Output should be similar to: `Terraform v0.14.9`

## Create Terraform configuration
Create a directory for the lab 1 files:
```sh
mkdir tf-lab1
cd $_
```
Terraform loads all files in the working directory that end in `.tf`.

Create a `main.tf` file with the following: 
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
  profile = "default"
  region  = "us-west-2"
}

resource "aws_instance" "lab1-tf-example" {
  ami           = "ami-830c94e3"
  instance_type = "t2.micro"

  tags = {
    Name = "Lab1-TF-example"
  }
}
```

This is a complete configuration that Terraform is ready to apply. In the following sections we'll review each block of the configuration in more detail.

## Providers 
Each Terraform module must declare which providers it requires, so that Terraform can install and use them. Provider requirements are declared in a `required_providers` block.

A provider requirement consists of a local name, a source location, and a version constraint:
The `provider` block configured the named provider, which in our case is `aws`. The provider is responsible for understanding the API and translating the Terraform configuration so it works with the vendor's API. Because Terraform has many providers you can represent almost any infrastructure type as a `resource` in Terraform.

You can optional specify a `profile` attribute in your provider block. This attribute refers to credentials stored in your AWS Configuration. 

**NOTE:** You should NEVER hard-code credentials into your `*.tf` files. 

## Resources 
The `resource` block defines infrastructure you want to create. This resource can be can be a physical component like an EC2 instance, GCP VM, or VMware machine, or it can be logical like a Heroku application. 

Our resource block has two strings before the block: a resource type, and resource name. 
In the above example, the type is `aws_instance` and the name is `lab1-tf-example`. This prefix of type maps to the provider. The `aws_instance` type tells Terraform it is managed by the `aws` provider.

We provide configuration data for our resource inside the resource block. These arguments are for things like machine size, image, VPC IDs, ssh username etc. 

## Initialize the directory
Now that we have our configuration file created run `terraform init` to download the providers used in our configuration.

Terraform uses a plugin-based architecture to support hundreds of infrastructure and service providers. Subsequent commands will use local settings and data during initialization.

Output will be similar to: 
```
Initializing the backend...

Initializing provider plugins...
- Finding latest version of hashicorp/aws...
- Installing hashicorp/aws v3.34.0...
- Installed hashicorp/aws v3.34.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

Terraform downloads the `aws` provider and installs it in a hidden subdirectory of the current working directory. The output shows which version of the plugin was installed.

Confirm the plugin was downloaded: 
```sh
sudo yum install -y tree 
tree .terraform/
```

You should see output like: 
```
.terraform/
└── providers
    └── registry.terraform.io
        └── hashicorp
            └── aws
                └── 3.34.0
                    └── linux_amd64
                        └── terraform-provider-aws_v3.34.0_x5
```

As you can see above Terraform download the aws provider from the official Hashicorp registry. 

## Format and validate configuration 
Terraform includes a `fmt` argument to ensure consistent formatting in files and modules written by different teams. It automatically updates configurations for easy readability and consistency. 

The `main.tf` file created is very basic, and is already formatted. 
```sh
terraform fmt 
```

If there were any formatting issues, like equal signs not aligned or wrong indentation `fmt` will fix them.

If any files are formatted Terraform will output the name of the file. If there are not formatting changes there is not output.

Now use the built-in `terraform validate` command to check and report any errors in modules, attribute names, and value types.

## Create infrastructure 
Now that the provider is downloaded, configuration files are formatted and validated it's time to create the infrastructure. 

Run `terraform plan` to review the changes Terraform will make. 

This output shows the execution plan, describing which actions Terraform will take in order to change real infrastructure to match the configuration.

Run `terraform apply`

The output has a + next to `aws_instance.lab1-tf-example`, meaning that Terraform will create this resource. Beneath that, it shows the attributes that will be set. When the value displayed is (known after apply), it means that the value won't be known until the resource is created.

Terraform will now pause and wait for your approval before proceeding. If anything in the plan seems incorrect or dangerous, it is safe to abort here with no changes made to your infrastructure.

In this case the plan is acceptable, so type `yes` at the confirmation prompt to proceed. Executing the plan will take a few minutes since Terraform waits for the EC2 instance to become available.





