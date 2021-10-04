# Packer & Terraform Capstone lab

## Overview
This lab will combine all of the things learned throughout this class. You will not be given step-by-step instructions and will need to do some research on your own. 

## Packer
Use what you've learned over the course of this class to create a packer image with a template that uses provisioners to install the `httpd` package. 

TIP: Start with the Amazon Linux AMI

### Bonus: Inject SSH key 
Create an SSH key pair, and use Packer to inject it into your image. 

## Terraform 
Setup Terraform to use remote state. 

Create a module that:
  - Uses a data source to create an EC2 instance from AMI you built previously with Packer.
  - Creates an Elastic Load Balancer 
  - Creates a VPC 
  - Creates a file in the build directory named `public_ips.txt` with the EC2 instance's public IP address.

You should have a VPC, with a Load Balancer, and at least one EC2 instance running.




 

 