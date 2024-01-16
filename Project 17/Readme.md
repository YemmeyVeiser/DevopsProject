##AUTOMATE INFASTRUCTURE WITH TERRAFFORM

I will replicate the below diagram using terraform .
![Architecture](./Project%2017/Images/Architecture%20Diagram.png)

The pre-requisite is to establish connectivity to AWS from our local machine by installing and configuring AWS CLI .
create a programmatic user on AWS called terraform and generate access keys that will be used on our local machine for configuration.
![terraform user](./Project%2017/Images/terraform_user_creation.png)

use ``aws configure`` command to configure aws cli connectivity to the AWS account.
![aws cli](./Project%2017/Images/aws_configure.png)

We will create a folder called PBL, and create a terraform file main.tf. inside the file we will create our resources as shown below.



````
provider "aws" {
  region = "eu-central-1"
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = "172.16.0.0/16"
  enable_dns_support             = "true"
  enable_dns_hostnames           = "true"
  enable_classiclink             = "false"
  enable_classiclink_dns_support = "false"

  tags{
    name = "Terraform VPC"
  }
}

````
now we will download the terraform providers and plugins by running the command ``terraform init``.

we can now run ``terraform plan`` and ``terraform apply`` to create our resources.
![Vpc](./Project%2017/Images/vpc_creation.png)

We will create the subnets for our architecture. we require 6 subnets according to the diagram.
2 public subnets for webservers.
2 public subnets for Data layer  
lets add the bloe cose to our main.tf file

``````

   #Create public subnets1
    resource "aws_subnet" "public1" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.0.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1a"

}

    #Create public subnet2
    resource "aws_subnet" "public2" {
    vpc_id                     = aws_vpc.main.id
    cidr_block                 = "172.16.1.0/24"
    map_public_ip_on_launch    = true
    availability_zone          = "eu-central-1b"
}

````````

Lets refactor our code by introducing variables and getting rid of hardcoded values as muxh as possible.

````

    variable "region" {
        default = "eu-central-1"
    }

    variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

    variable "enable_dns_support" {
        default = "true"
    }

    variable "enable_dns_hostnames" {
        default ="true" 
    }

        provider "aws" {
    region = var.region
    }

    # Create VPC
    resource "aws_vpc" "main" {
    cidr_block                     = var.vpc_cidr
    enable_dns_support             = var.enable_dns_support 
    enable_dns_hostnames           = var.enable_dns_support
    

    }
````
We will make use of terraform ``data sources`` to get the number of availability zones available.

````
        # Get list of availability zones
        data "aws_availability_zones" "available" {
        state = "available"
        }

``````
lets create a count argument where we assigned the number of subnets we intend to create.
````
    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = "172.16.1.0/24"
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }


````
we will make the cidr_block dynamic by using a function ``cidrsubnet()``that accepts 3 parameters. 
````
    # Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = 2
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }


````
lastly we will remove the hardcoded count value by introducing the length function``length()`` which can be used to determine the length of a given list, map or string.

since out data source gives the list of availability zones, we can pass it into the length function ``length()`` to dynamically get the nymber of AZs.
````
# Create public subnet1
    resource "aws_subnet" "public" { 
        count                   = length(data.aws_availability_zones.available.names)
        vpc_id                  = aws_vpc.main.id
        cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
        map_public_ip_on_launch = true
        availability_zone       = data.aws_availability_zones.available.names[count.index]

    }

```` 
Going by our architecture we only need to create 2 public subnets, so we will assigned a variable to the number of subnets we intend to create as shown below.
````
  variable "preferred_number_of_public_subnets" {
      default = 2
}

````
we will provide the count argument with a condition to check and use the value assigned to the desired number of subnet variable, if not it will use the value returned by the length function.
````
# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}

````
Now the entire code should look like the below


````
# Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = 2
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}


# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]

}


````
## variables.tf & variables.tfvars
we can make the code more readable and structured by introducing a variables file  and moving all the variables to this file.

- create a file in the PBL folder and name it ``variables.tf`` 
- create another file and name it ``variable.tfvars`` this will have the values assigned for all variables declared.
  ![variable file](./Project%2017/Images/variables_file.png)

  each configuration file should look like the below.
  ### Main.tf

````
  # Get list of availability zones
data "aws_availability_zones" "available" {
state = "available"
}

provider "aws" {
  region = var.region
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block                     = var.vpc_cidr
  enable_dns_support             = var.enable_dns_support 
  enable_dns_hostnames           = var.enable_dns_support
  enable_classiclink             = var.enable_classiclink
  enable_classiclink_dns_support = var.enable_classiclink

}

# Create public subnets
resource "aws_subnet" "public" {
  count  = var.preferred_number_of_public_subnets == null ? length(data.aws_availability_zones.available.names) : var.preferred_number_of_public_subnets   
  vpc_id = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4 , count.index)
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[count.index]
}

  ````
  #### variables.tf

  ````
  variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = null
}

  ````

#### variables.tfvars

````
variable "region" {
      default = "eu-central-1"
}

variable "vpc_cidr" {
    default = "172.16.0.0/16"
}

variable "enable_dns_support" {
    default = "true"
}

variable "enable_dns_hostnames" {
    default ="true" 
}

variable "enable_classiclink" {
    default = "false"
}

variable "enable_classiclink_dns_support" {
    default = "false"
}

  variable "preferred_number_of_public_subnets" {
      default = null
}

````

Now we can run ``terraform plan`` and then ''terraform apply`` and ensure the resources are created.
as shown below, our resources were successfully created.
![Resource creation](./Project%2017/Images/creation_complete.png)