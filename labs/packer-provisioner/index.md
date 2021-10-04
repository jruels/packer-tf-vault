# Packer Lab 2

## Lab objectives: 
* Introduce provisioners
* Update template to use new provisioners
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
  # This example demostrates that you can load variables from the environment.
  # In real usage, the AWS sdk used by the Amazon builder can automatically
  # load credentials from the environment, so you wouldn't need to do this
  # step. See the Packer AWS docs for more details.
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
  ami_name      = "packer-linux-aws-demo"
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


