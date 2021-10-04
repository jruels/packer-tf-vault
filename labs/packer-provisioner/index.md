# Packer Lab 2

## Lab objectives: 
* Introduce provisioners
* Update template to use new provisioners   
* Update template to use variables

### Using provisioners   
Enter your working directory: 
```sh
cd ~/$(date +%Y%m%d)/packer
```
Packer can customize your image using provisioners. This lab is going to introduce you to the `file` and `shell` provisioners.

Create a file named `welcome.txt` and populate it with: 
```
WELCOME TO PACKER!
```

Create a file named `example.sh` and add: 
```sh
#!/bin/bash
echo "hello"
```

Add execute permissions to `example.sh`
```sh
chmod +x example.sh
```

Create a template named `firstrun.pkr.hcl` and populate with:
```hcl
variable "ami_name" {
  type = string
  default = "${env("CUSTOM_AMI_NAME")}"
}

variable "region" {
  type    = string
  default = "us-east-1"
}

# source blocks are generated from your builders; a source can be referenced in
# build blocks. A build block runs provisioners and post-processors on a
# source.
source "amazon-ebs" "firstrun" {
  ami_name      = "packer-linux-aws-redis"
  instance_type = "t2.micro"
  region        = "${var.region}"
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/*ubuntu-xenial-16.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["099720109477"]
  }
  ssh_username = "ubuntu"
}

# a build block invokes sources and runs provisioning steps on them.
build {
  sources = ["source.amazon-ebs.firstrun"]

  provisioner "file" {
    destination = "/home/ubuntu/"
    source      = "./welcome.txt"
  }
  provisioner "shell" {
    inline = ["ls -al /home/ubuntu", "cat /home/ubuntu/welcome.txt"]
  }
  provisioner "shell" {
    script = "./example.sh"
  }
}
```

### Build the image
To take advantage of the "env" function embedded in our default `ami_name` variable, we can set the env var referenced; then we don't have to pass the -var flag to the `packer build` command. 

```sh
export CUSTOM_AMI_NAME="firstrun-provisioner"
packer build firstrun.pkr.hcl
```

If you want to use a `source_ami` instead of a `source_ami_filter` it may look something like this:
```
source_ami = "ami-fceotj67
```

After running your `packer build` you should see output stating the image was created successfully. Review the output and confirm the provisioners ran without error.

### Install packages
We'll use the built-in shell provisioner that comes with Packer to install Redis. Modify the `example.pkr.hcl` template we made in the previous lab and add the following provisioner block inside of the already-existing build block. We'll explain the various parts of the new configuration following the code block below.
```hcl
build {
  sources = ["source.amazon-ebs.example"]

  provisioner "shell" {
    inline = [
      "sleep 30",
      "sudo apt-get update",
      "sudo apt-get install -y redis-server",
    ]
  }
}
```

**NOTE:** The `sleep 30` in the example above is very important. Because Packer is able to detect and SSH into the instance as soon as SSH is available, Ubuntu actually doesn't get proper amount of time to initialize. The sleep makes sure that the OS properly initializes.

**Note:** This example does not create a source block for you; the source was set up in the previous lab, and should be defined above the build block in this example.

By default, each provisioner is run for every builder defined. So if we had two builders defined in our template, such as both Amazon and DigitalOcean, then the shell script would run as part of both builds. 

The one provisioner we defined has a type of shell. This provisioner ships with Packer and runs shell scripts on the running machine. In our case, we specify two inline commands to run in order to install Redis.

### Build
With the provisioner configured, give it a pass once again through `packer validate` to verify everything is okay, then build it using `packer build example.pkr.hcl`. The output should look similar to when you built your first image, except this time there will be a new step where the provisioning is run.

The output from the provisioner is too verbose to include in this tutorial, since it contains all the output from the shell scripts. But you should see Redis successfully install. After that, Packer once again turns the machine into an AMI.

If you were to launch this AMI, Redis would be pre-installed.

### Update variables

In this section you will modify the template to generate a unique AMI name.

Add the following variable block to your template   
```hcl
variable "ami_prefix" {
  type    = string
  default = "packer-linux-aws-redis"
}
```

Add a local variable to your template   
```hcl
locals {
  timestamp = regex_replace(timestamp(), "[- TZ:]", "")
}

```

Local blocks declare the local variable name (`timestamp`) and the value (`regex_replace(timestamp(), "[- TZ:]", "")`). You can set the local value to anything, including other variables and locals. Locals are useful when you need to format commonly used values.

In this example, Packer sets the `timestamp` local variable to a formatted timestamp using functions.

Inputs variables and local variables are constants — you cannot update them during run time.


Update AMI name

In your Packer template, update your source block to reference the `ami_prefix` variable. Notice how the template references the variable as `var.ami_prefix`.

```hcl
source "amazon-ebs" "ubuntu" {
-  ami_name      = "packer-linux-aws-redis"
+  ami_name      = "${var.ami_prefix}-${local.timestamp}"
   ## ...
}
```

### Build image

Build the image.
```sh
packer build firstrun.pkr.hcl
```


After running your `packer build` you should see output stating the image was created successfully. Notice how the Packer creates an AMI where its name consists of `packer-linux-aws-redis`, the default value for the `ami_prefix` variable, and a timestamp.

Build image with variables 

Since `ami_prefix` is parameterized, you can define your variable before building the image. There are multiple ways to assign variables. The order of ascending precedence is: variable defaults, environment variables, variable file(s), command-line flag. In this section, you will define variables using variable files and command-line flags.

#### Build image with variable file
Create a file named `firstrun.pkrvars.hcl` and add the following snippet into it.
```hcl
ami_prefix = "packer-aws-redis-var"
```

Build the image with the `--var-file` flag.
```sh
packer build --var-file=firstrun.pkrvars.hcl firstrun.pkr.hcl
```

Notice how the AMI name starts with `packer-aws-redis-var-`, the value for `ami_prefix` defined by the variable file.

Packer will automatically load any variable file that matches the name `*.auto.pkrvars.hcl`, without the need to pass the file via the command line.

Rename your variable file so Packer automatically loads it.

```sh
mv firstrun.pkrvars.hcl firstrun.auto.pkrvars.hcl
```

Build the image and notice how the AMI name starts with `packer-aws-redis-`. The `packer build .` command loads all the contents in the current directory.

#### Build image with command-line flag
Build the image, setting the variable directly from the command-line.

```sh
packer build --var ami_prefix=packer-aws-redis-var-flag .
```

Notice how the AMI name starts with `packer-aws-redis-var-flag`, the value for `ami_prefix` defined by the command-line flag. The command-line flag has the highest precedence and will override values defined by other methods.


Bonus: Configure `region` in a variables file.

## Congrats! 
