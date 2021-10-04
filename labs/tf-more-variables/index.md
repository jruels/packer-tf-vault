# Terraform Lab 3

## Overview 
In this lab, you will use Terraform to deploy a web application on AWS. The infrastructure will include a VPC, load balancer and EC2 instances. 

Input variables make Terraform configurations more flexible by defining values that can be set by users. You will parameterize this configuration with Terraform input variables. 

## Setup lab files 
Under our working directory create a `tf-lab3` directory:
```sh
mkdir tf-lab3 
cd $_
```

In the new working directory clone the GitHub repository:

```sh
git clone https://github.com/jruels/learn-terraform-variables.git
```

Enter the directory: 
```sh
cd learn-terraform-variables
```

The configuration in `main.tf` defines a web application, including a VPC, load balancer, and EC2 instances.

Review the configuration files, and pay attention to the resources being created and their hard-coded values.

Initialize this configuration.
```sh
terraform init
```

Now apply the configuration.
```sh
terraform apply
```

The infrastructure will be created,  but we want our code to be reusable. 

## Using Parameters

You can define variables anywhere in your configuration files, but the recommended approach is to declare them in a `variables.tf` file. This is the standard and makes it easier for users to understand how the configuration should be customized. 

To parameterize an argument with an input variable, you will first define the variable in `variables.tf`, then replace the hard-coded value with a reference to that variable in your configuration.

Add a block declaring the `aws_region` variable to `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}
```

Our variable block has three optional arguments. 

- Description: A short description to document the variables purpose 
- Type: The type of data contained in the variable.
- Default: The default value of the variable.

It is recommended to set a description and type for all variables. When practical, you should also set a default value.

If you do not set a default value, you must assign a value before Terraform can apply the configuration.

Terraform does not support unassigned variables. You will see some of the ways to assign values to variables later in this lab.

## Doing things on your own
The rest of this lab uses principles taught previously, and will not provide step-by-step instructions. Please work to solve the lab yourself, but if you get stuck reach out to the instructor.

You can refer to variables in your configuration with `var.<variable_name>`.

Edit the provider block in `main.tf` to use the new `aws_region` variable.

Add a declaration for the `vpc_cidr_block` variable to `variables.tf` with the following: 
- variable name: `vpc_cidr_block`
- description: `CIDR block for VPC`
- type: `string`
- default: `10.0.0.0/16`

Now, replace the hard-coded value for the VPC's CIDR block with a variable in `main.tf`.

Apply the updated configuration. The default values of these variables are the same as the hard-coded values they replaced so no changes will be made.

## Create multiple instances 
Use a `number` type to define how many instances are supported by this configuration. 

Add a variable block to `variables.tf` with the following: 
- variable name: `instance_count`
- description: `Number of instances to provision.`
- type: number 
- default: `2`

Update the EC2 instances resource to use the `instance_count` variable in `main.tf`

Now Terraform will convert the values into the correct type. The `instance_count` variable would also work using a string ( "2" ) instead of number ( 2 ). 


Once again the variables added have the same values as the original hard-coded values. Run `terraform apply` and you'll see it does not need to make any changes.

