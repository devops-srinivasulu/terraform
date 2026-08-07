in terraform we use varibles to prevent the hardcoding the values which we are taking like sensitive . 
for that create one main.tf, variable.tf i same working directory .

main.tf

```
provider "aws" {
  region = "us-east-1"

}
data "aws_ami" "ubuntu" {
  most_recent = true

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["099720109477"] # Canonical
}
resource "aws_instance" "example" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type    ## if tey use list you can use "instance_type = var.instance_type[0]" like that to find first list of value .
  key_name      = var.key_name # my keypare name 
  tags = {
    Name = var.instance_name #this ismy actually instance name 
  }
}

```

Variable.tf

```
variable "instance_type" {
  type    = string
  default = "t2.micro"

}

variable "key_name" {
  type    = string
  default = "jenkins"
}
variable "instance_name" {
  type    = string
  default = "variable"

}
````

so while we use "terraform apply -auto-approve" then the flow is 
```
1. Read variables
        ↓
2. Read provider
        ↓
3. Read data sources
        ↓
4. Create resources

```
this is for only single environment  what if for suppose we have to create infra for multi env then we should use .tfvar files
===================================================================================
.tfvars file:  -> it is used to eperate the values of environment ... we no need to change any maintf files when we should have same setup for all env .

## for example we are going to create infra for two environments (prod, dev) the file strucure is 

```
terraform_project/
|
|__ main.tf
|__ variable.tf
|__dev.tfvars ## dev relaated values are here example: i am going to keepo instacne type "t2.micro" for dev env
|
|__ prod.tfvars  ## prod related values are here example: i am going to keep instance type "t2.medium" for prod env

```
## variable.tf 

```
variable "instance_type" {}
variable "key_name" {}
variable "instance_name" {}
```

## dev.tfvars
```
instance_type = "t2.micro"   ## if you wann use list you can use like "instance_type = ["t2.micro", "t2.medium"]"
key_name      = "jenkins"
instance_name = "dev-server"

```

## prod.tfvars
```
instance_type = "t2.medium"   ## if you wann use list you can use like "instance_type = ["t2.micro", "t2.medium"]"
key_name      = "jenkins"
instance_name = "prod-server"
```

so now while you are going to create infra for dev environment use this commands
```
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars" -auto-approve

## for destroy 
terraform destroy -var-file="dev.tfvars" -auto-approve
```
if you are going to create infra for prodution environment
```
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars" -auto-approve

## for destroy 
terraform destroy -var-file="prod.tfvars" -auto-approve
```
==============================================================



for supose we are going to  create prod infra(terraform apply -var-file="prod.tfvars") after creating dev infra(terraform apply -var-file="dev.tfvars").. then it first desttry the dev infra then create the prod  one .. if the production its not a suffient method .. when we should keep both dev, prod infra . to aciecve that we have "workspace" concept .

WORKSPACE: it's allow us to isolate  environments (dev,prod...etc) . 

commands::
```
1 .terraform workspace list     ## to list the workspaces which we have including current workspace with * mark  

2. terraform workspace show    ## to see current workspace

3. terraform workspace new <name of the workspace>   ## to crate new work space ex:   terraform workspace new dev      it automatically create new one and switch to that workspace >

4. terraform workspace select <name of the workspace>  ## to switch perticular workspace ex: terraform workspace select dev

5. terraform workspace delete <name of workspace> ## to delete the perticular worksapce ex: dev 
```

if you wanna create dev infrastructure follow these:
```
  terrafrom workspace new dev
  ## to confirm current workspace is dev
  terraform worksapce list  #or  terraform worksapce show
  terraform apply -var-file="dev.tvars" -auto-approve 

```
 same for prod also .

 if you wanna destroy  dev infra which is in dev workspace.

 ```
# first confirm you are under dev workspace if you are under another worksapce switchto dev .
terraform workspace select dev
# confirm the worksapce by using this command 
terraform workspace show   ## you should see dev
## now destroy 
terraform destroy -var-file="dev.tfvars" -auto-approve    # to destroy the infra which is crated under this workspace dev . 
 
  ```
  smae for prod also .
  ===> in this way we isolate enviromnets by using workspace and .tfvars 

