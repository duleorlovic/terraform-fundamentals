---
layout: post
---

Terraform provides Configuration Management on an infrastructure level, not on
the level of software on your machines.

# Install

Download binary https://www.terraform.io/downloads
Example https://github.com/wardviaene/terraform-course

```
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -install-autocomplete
```

# Usage

Commands

```
# download provider plugins
terraform init
# Terraform has created a lock file .terraform.lock.hcl to record the provider
# selections it made above. Include this file in your version control repository
git init .
echo .terraform/ >> .gitignore

# check the changes live
terraform plan
# you can save to a file and apply them
terraform plan -out changes.terraform && terraform apply changes.terraform

# push the changes
terraform apply
terraform apply -auto-approve # accept yes
# I suggest to save the state in git
terraform apply -auto-approve && git add .

# correct the format
terraform fmt
# validate syntax
terraform validate

# inspect current state from terraform.tfstate
terraform show

# to show all resources from current state
terraform state list
# to rename resource
terraform state mv aws_instance.example aws_instance.production

# create visual representation of a configuration or execution plan
terraform graph

# find resource ID and import to the state as ADDRESS. Only for importing the
# state still need to write resource definitions since next apply will remove it
# todo: https://learn.hashicorp.com/tutorials/terraform/state-import
terraform import ADDRESS ID
terraform import aws_instance.example i-0dba323asd123

# show defined outputs
terraform output
terraform output resource-name

# refresh remote state
terraform refresh

# configure remote state storage
terraform remote

# to remove
# this will clear the state file but you can restore using git or
# terraform.tfstate.backup
terraform destroy

# manually mark resource as tainted, it will be destructed and recreated at the
# next apply
terraform taint
terraform untaint
```

main.tf for `terraform` block (for main setting and providers)
https://registry.terraform.io/ and for resources

```
# main.tf
terraform {
  required_providers {
    # https://www.terraform.io/language/providers/requirements
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 2.13.0"
    }
  }
}
```

```
# resource.tf
# provider block use local name of the provider
provider "docker" {}

# resource block has two strings before block: resource type and resource name
# prefix for resource type match the provider name. resource id is type.name
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

# access the container on http://localhost:8000/
resource "docker_container" "nginx" {
  image = docker_image.nginx.latest
  name = "tutorial"
  ports {
    internal = 80
    external = 8000
  }
}
```

variable block
https://learn.hashicorp.com/tutorials/terraform/variables
```
# variables.tf or var.tf
# without value or default will be asked
variable "AWS_ACCESS_KEY" {}

variable "myvar" {
  type = "string"
  default = "hello"
}

# array
variable "mylist" {
  description = "A list of zones"
  type = list(string)
  default = [ "abc", "qwe" ]
}

variable "mymap" {
  type = map(string)
  default = {
    mykey = "my value"
  }
}

# tuple is like a list with a different type
[0, "string", false]

# you can override variable as attribute:
terraform apply -var "myvar=NewName" -var "RDS_PASSWORD=$PASSWORD"_

# you can test inside `terraform console`
to create:
tomap({"k"="v", "k2"="v2"}) # returns { "k"="v", "k2"="v2" }
tolist("a", "b") # returns array ["a", "b"]

var.myvar
"${var.myvar}" # interpolation
var.mymap["mykey"]
values(mymap) # ["my value"]
lookup(var.mymap, "mykey", "default")
var.mylist[0]
element(var.mylist, 0)
index(mylist, elem) # find index of element in a mylist
slice(var.mylist, 0, 2)
merge(map1, map2) # merge two arrays
coalesce(string1, string2) # returns first non empty value
format("server-%03d", count.index + 1)  returns server-001 server-002
join(delim, mylist), join(",", var.AMIS) # ami-1,ami-2
replace("aaab", "a", "c") # "cccb"
split(",", "a,b,c") # ["a", "b", "c"]
substring("abcd", offset, length)
timestamp()
uuid()
```

Math `${2+3*4}`

Count
```
count.FIELD
$[count.index]
```

Conditionals
```
count = "${var.env == "prod" ? 2 : 1 }"
```

For loops
```
[for s in ["a", 1]: upper(s)]
{for k,v in { "a"=1 }: k => v}
```

Nested block does no have equal sign and we can repeat them
```
  ingress {
  }
  ingress {
  }
```
For each loop uses `dynamic` and `content` keywords and `.key` and `.value`
attributes of the variable
```
  dynamic "ingress" {
    for_each = [22, 443]
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
    }
  }
```

# Project structure

```
staging/
production/
modules/
```
to differentiate between accounts you can use different profiles from aws
configure or different .env files

Secrets are usually stored in `terraform.tfvars` (autoloaded, other name can be
loaded like `-var-file="testing.tfvars"`) which should be git ignored
```
echo */terraform.tfvars >> .gitignore
```
and contains all keys in format `NAME = "value"`
```
AWS_ACCESS_KEY = "AKIA..."
AWS_SECRET_KEY = "6JX..."
AWS_REGION = "us-east-1"
```
and those variables should be declared in `var.tf` (note that they can be found
in tfstate so keep tfstate in secure place also if you use secrets)
```
echo */terraform.tfstate >> .gitignore
echo */terraform.tfstate.backup >> .gitignore
```
You can use env to define variables when you export prefix TF_VAR environment
variable `export TF_VAR_DB_PASSWORD=asd1234`.
So use it instead of using `terraform.tfvars`
```
# var.tf
# without value or default will be asked
# variable "AWS_ACCESS_KEY" {}
# variable "AWS_SECRET_KEY" {}
variable "AWS_REGION" {
  default = "us-east-1"
}

variable "AMIS" {
  type = map
  default = {
    # find ami on https://cloud-images.ubuntu.com/locator/ec2/ search example
    # us-east-1 hvm
    # https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html#finding-quick-start-ami
    us-east-1 = "ami-0b0ea68c435eb488d"
  }
}

variable "DB_PASSWORD" {
  description = "RDS posgres user password"
  sensitive   = true
}
```

To output which was marked as sensitive you can
* `t output DB_PASSWORD` explicitly show that value
* `grep --after-context=10 outputs terraform.tfstate` grep state
* decode plan that is saved in tmp folder
  ```
  t plan -target=vault_generic_secret.foobar -out=/tmp/tfplan
  t show -json /tmp/tfplan  | jq -r ".resource_changes[0].change.before.data , .resource_changes[0].change.after.data"
  ```
* mark as non sensitive
  ```
    output "mysecret" {
      value = nonsensitive(var.mysecret)
    }
  ```

aws provider

```
# provider.tf
provider "aws" {
  # comment so current `aws configure` will be used
  access_key = var.AWS_ACCESS_KEY
  secret_key = var.AWS_SECRET_KEY
  region     = var.AWS_REGION
}
```

# Provision software

Use file uploads with `file` provisioner and `remote-exec` to run the script
```
provisioner "file" {
  source = "app.conf"
  destination = "/etc/myapp.conf"
  connection {
    type = ssh
    user = var.instance_username
    password = var.instance_password
  }
}
```

For aws we use ssh keypairs for which we need resource `aws_key_pair`

```
# keys.tf
# key is generated with ssh-keygen -f mykey
# https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/key_pair
resource "aws_key_pair" "mykeypair" {
  # existing keys can be imported with: terraform import aws_key_pair.deployer deployer-key
  # https://console.aws.amazon.com/ec2/v2/home?region=us-east-1#KeyPairs:
  # mykeypair will be destroyed when we run terraform destroy
  # Find IP address from aws console or using output
  # ssh -i mykey ubuntu@3.84.117.126
  # ssh -i mykey ubuntu@$(cat terraform.tfstate | jq -r '.resources[].instances[].attributes.public_ip | select( . != null )')
  # ssh -i mykey ubuntu@$(aws ec2 describe-instances --query "Reservations[*].Instances[*].PublicIpAddress" --output=text)
  key_name = "mykeypair"
  public_key = file(var.PATH_TO_PUBLIC_KEY)
}

# resource.tf
resource "aws_instance" "example" {
  ami           = lookup(var.AMIS, var.AWS_REGION)
  instance_type = "t2.micro"
  key_name = aws_key_pair.mykeypair.key_name

  provisioner "file" {
    source = "script.sh"
    destination = "/tmp/script.sh"
  }
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/script.sh",
      "sudo /tmp/script.sh"
    ]
  }
  # another way to output info is using local-exec provisioner (it is performed
  # only first time resource is created)
  provisioner "local-exec" {
    command = "echo ${aws_instance.example.private_ip} >> private_ips.txt"
  }
  connection {
    user = var.INSTANCE_USERNAME
    private_key = file(var.PATH_TO_PRIVATE_KEY)
    host = self.public_ip
  }
}
output "ip" {
  description = "Public IP address for EC2 instance"
  value = aws_instance.example.public_ip
}
output "private_ip" {
  value = aws_instance.example.private_ip
}
```
Add to var
```
# var.tf
variable "PATH_TO_PUBLIC_KEY" {
  default = "mykey.pub"
}

variable "PATH_TO_PRIVATE_KEY" {
  default = "mykey"
}

variable "INSTANCE_USERNAME" {
  default = "ubuntu"
}
```
```
# script.sh
#!/bin/bash
apt-get update
apt-get -y install nginx
```

## State

`terraform.tfstate` is where it keeps track of remote state. there is also
`terraform.tfstate.backup` for previous state.
You can keep `terraform.tfstate` in git repo so you can see state changes.
For example when you remove the instance from aws console, `terraform apply`
will make a changes to meet the correct remote state.
But is is not adviceable since another person can change the state so you loose
the sync since you do not use same lock (also it contains sensitive information)
For single user repo you can add both `terraform.tfstate` and
`.terraform.lock.hcl` to the repo. Lock is important so we use same versions of
the providers and modules.

```
# Find all resource names
cat terraform.tfstate | jq '.resources[].name'
# find public_ip
cat terraform.tfstate | jq '.resources[].instances[].attributes.public_ip | select( . != null )'
```

You can save state remote using a backend functionality in terraform
https://www.terraform.io/language/settings/backends/remote

Credentials are different then those secrets for provision.
```
# backend.tf
terraform {
  backend "s3" {
    bucket = "duleorlovic-test"
    key = "tfstate"
    region = "eu-central-1"
  }
}
```
and run `terraform init` to check backend.
Inside bucket properties you can enable "Versions" to see old states.

## Packer

https://learn.hashicorp.com/packer you can create a custom image
todo: https://learn.hashicorp.com/collections/terraform/provision

# Other providers

https://registry.terraform.io/browse/providers
Any company that opens API can be an provider: Datadog (monitoring), Github,
mailgun, DNSSimple

Providers are similar.

# Template provider templatefile

For creating customized configuration files, template based on variables from
resource attributes (for example public ip address).
Use case is cloud init config (user-data on AWS).

```
# templates/init.tpl
echo "hello world ${myip}"
```
Instead of separate resource
```
# deprecated
data "template_file" "my-template" {
  template = file("templates/init.tpl")

  vars {
    myip = aws_instance.database1.private_ip
  }
}

# usage
resource "aws_instance" "web" {
  user_data = data.template_file.my-template.rendered
}
```
but now we use `templatefile(file, map)` function
```
locals {
  web_vars = {
    my_ip = aws_instance.database1.private_ip
  }
}

resource "aws_instance" "web" {
  user_data = templatefile("templates/init.tpl", local.web_vars)
  # or inline
  user_data = templatefile("templates/init.tpl", {
    my_ip = aws_instance.database1.private_ip
  })
}
```

# Modules

You can organize files inside folders (locally) or remote on github.
```
module "module-example" {
  source = "github.com/duleorlovic/terraform-module-example"
  # locally
  source = "./module-example"

  # override or add additional arguments
  region = "us-east-1"
}
```
For example
```
# module-example/var.tf
variable "region" {} # the input parameter
variable "ip-range" {}

# module-example/cluster.tf
resource "aws_instance" "instance-1" {
}

# module-example/output.tf
output "aws-cluster" {
  value = aws_instance.instance-1.public_ip
}
```
To use output of module you can reference `module.module-name.output`
```
output "some-output" {
  value = module.module-example.aws-cluster
}
```
You have access to `path.cwd` `path.module` and `path.root`.
To download you need to run
```
terraform get
ls -l .terraform/modules
```

todo: https://learn.hashicorp.com/tutorials/terraform/module

# Terraform Cloud

quick start
```
terraform login
git clone https://github.com/hashicorp/tfc-getting-started.git
cd tfc-getting-started/
scripts/setup.sh
```
todo: https://learn.hashicorp.com/collections/terraform/cloud-get-started
https://github.com/evilmartians/terraforming-rails/tree/master/tools/lint_env
