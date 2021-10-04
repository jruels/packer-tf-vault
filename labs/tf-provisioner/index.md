# Terraform Lab 6

## Overview 
In this lab you will update `main.tf` to include provisioners.

Provisioners allow you to run shell scripts on the local machine, or remote resources. You can also use vendor provisioners from Chef, Puppet, and Salt.

## Working directory 
This lab updates the `main.tf` in the `tf-lab4` directory. 

Add a `local-exec` provisioner with the following attributes: 
- command: Echo the public IP addresses into a file named `public_ips.txt`

Add another `local-exec` provisioner with the following attributes: 
- command: Echo the private IP addresses into a file named `private_ips.txt`

 