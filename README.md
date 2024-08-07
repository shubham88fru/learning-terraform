# learning-terraform

- Terraform is a IaaC solution. It works for almost every cloud
  provider we can think of. Unlike, AWS cloudformation which is AWS
  specific and uses JSON for defining the template, Terraform can be
  used for creating infrastructure/resources on various cloud providers and uses a more readable language (Hashicorp)

- When creating a resource in terraform, the resource always
  begins with the name of provider. E.g. "aws_s3", "google_folder" etc.

- The format to refer the output of a resource is
  `<resource_type>.<resource_identifier>.<attribute_name>`

- on running `terraform init`, terraform actually downloads the
  provider binary that we are using in our tf files. Providers (aws, azure etc) are separate binaries to terraform and are hosted
  by Hashicorp. They are downloaded in a folder named `.terraform` in the workspace where we ran the `terraform init` command.

- `alias` is a property that we can set on any provider block (It's) not specific to aws. It gives us a way to distinguish between two or more providers. If we define two or more instances of same provider, every definition (after the first) must have an alias set.

- Every resource has a provider property that we can set. The format of the value is set by `<provider_name>.<provider_alias>`. E.g. aws.ohio

```terraform
//Terraform works with many cloud providers.
//Here we're saying that we need to created
//resources on aws and in oregon region.
provider "aws" {
    region = "us-west-2"
}

//Define the resources to be created.
//Here, we are defining a s3 bucket to be created.
resource "aws_s3_bucket" "first_tf_bucket" {
    bucket = "shubham-first-tf-bucket"
}
```

```terraform
//STEP 1: Initialize terraform
terraform init

//STEP 2: Terraform apply/plan
terraform plan //gives a gitst of what tf will do to acheive state.
terraform apply //create the resources.

//STEP 3: Destroy the created resource with one command.
terraform destroy //destroys the resources create via tf file.
```

```tf
//Interpolation/using resource outputs.
//create a vpc resource.
resource "aws_vpc" "my_vpc" {
    cidr_block = "10.0.0.0/16"
}

//Note we are using the output of vpc created above to get the id.
resource "aws_security_group" "my_security_group" {
    vpc_id = aws_vpc.my_vpc.id
    name = "Example security group"
}

//we use the security group resource created above
//to get the id.
resource "aws_security_group_rule" "tls_in" {
    protocol = "tcp"
    security_group_id = aws_security_group.my_security_group.id
    from_port = 443
    to_port = 443
    type = "ingress"
    cidr_blocks = ["0.0.0.0/0"]
}
```

```terraform
//enforcing a particular version of provider to be used
provider "aws" {
  region = "us-east-2"
}

terraform {
  required_providers {
    aws = {
      version = "~>3.46" //ensure version >= 3.46 but <= 4.0
    }
  }
}
```

```terraform
//multiple instances of same provider in a single file.
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  region = "us-east-2"
  alias = "ohio" //a different region. Alias will be used to choose.
}

//this resource will be created in us-east-1, by default.
resource "aws_vpc" "n_virginia_vpc" {
  cidr_block = "10.0.0.0/16"
}

//this resource will be created in ohio (us-east-2)
resource "aws_vpc" "ohio_vpc" {
  cidr_block = "10.1.0.0/16"
  provider = aws.ohio //chose ohio
}
```

- A data source in terraform is used to fetch data from a resource that is not managed by the current terraform project. This allows it to be used in the current project. Once referenced in the terraform file, the data source can be accessed as `data.<data_type>.<data_identifier>.<attribute_name>`.

```terraform
provider "aws" {
  region = "us-east-2"
}

data "aws_s3_bucket" "bucket" {
  bucket = "my-already-existing-bucket"
}

resource "aws_iam_policy" "my_bucket_policy" {
  name = "my-bucket-policy"
  policy = <<EOF //start of a multi line string
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": [
          "s3:ListBucket"
        ],
        "Effect": "Allow",
        "Resource": [
          "${data.aws_s3_bucket.bucket.arn}" //since we're in a string, use interpolation to evaluate.
        ]
      }
    ]
  }
  EOF //end of multi line string.
}
```

```terraform
//a terraform output block.
//outputs the value on to the console when `terraform apply`
//output blocks are more like a javascripts `console.log`?
output "message" {
  value = "Hello, world!"
}

//outputing resource properties.
provider "aws" {
  region = "us-east-2"
}

resource "aws_s3_bucket" "first_bucket" {
  bucket = "my-bucket"
}

output "bucket_name" {
  value = aws_s3_bucket.first_bucket.id
}

output "bucket_arn" {
  value = aws_s3_bucket.first_bucket.arn
}

output "bucket_information" {
  value = "bucket name: ${aws_s3_bucket.first_bucket.id}, bucket arn: ${aws_s3_bucket.first_bucket.arn}"
}
```

```terraform
//'local' in terraform is like variables in a normal programming language. Confusingly , terraform also has a concept of a variable which is basically an input.
//a local block is defined with `locals` keyword.

provider "aws" {
  region = "eu-west-1"
}

locals {
  first_part = "hello"
  second_part = "${local.first_part}-there"
  bucket_name = "${local.second_part}-how-are-you-today"
}

resource "aws_s3_bucket" "bucket" {
  bucket = local.bucket_name
}
```

```terraform
//file(...) - used to reference a file, it sets the value of the variable to the contents of the file.
//main.tf
provider "aws" {
  region = "eu-west-1"
}

resource "aws_iam_policy" "my_bucket_policy" {
  name = "list-buckets-policy"
  policy = file("./policy.iam")
}

//policy.iam
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:ListBucket"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

```terraform
//templatefile(...) - this function allows us to define placeholders in a template file and then pass their values at runtime.
//main.tf
locals {
  rendered = templatefile("./example.tpl", { name = "kevin", number = 7})
}

output "rendered_template" {
  value = local.rendered
}

//example.tpl
hello there ${name}. There are ${number} things to say.
```

```terraform
//A variable in terraform is something that can be set at runtime. It allows you to vary what Terraform will do by passing in or using a dynamic value. Variables are basically used to take user inputs.

provider "aws" {
  region = "us-east-2"
}

variable "bucket_name" {
  description = "the name of the bucket you wish to create"
}

resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket_name //a variable is referenced using `var`
}
```

```bash
#A way to supply value for variables through command line.
terraform apply -var bucket_name=first_bucket -var bucket_suffix=foo

#Supply values for variables through env variable.
#To do this, the env variables should be named per this convention
#TF_VAR_<variable_identifier>
export TF_VAR_bucket_name=first_bucket
export TF_VAR_bucket_suffix=fluff
```

```terraform
//Another way to set values for variables is through a file named
//`terraform.tfvars`. It is a special file name that terraform looks at for values of variables.

//if we want a custom name for the file, name then with an extension `.auto.tfvars`.

//e.g. terraform.tfvars file
bucket_name="first_bucket"
bucket_suffix="fluff"
```

```terraform
//more  complex variables
//main.tf
variable "instance_map" {}
variable "environment_type" {}

output "selected_instance" {
  value = var.instance_map[var.environment_type]
}

//terraform.tfvars
instance_map = {
  dev = "t3.small"
  test = "t3.medium"
  prod = "t3.large"
}

environment_type = "dev"
```

```terraform
//variable type constraints
//basic types: string, bool, number
//complext types: list(<TYPE>), set(<TYPE>), map(<TYPE>), object(), type([<TYPE>, ...])
variable "a" {
  type = string
  default = "foo"
}

variable "b" {
  type = bool
  default = true //could assing any truthy or falsy.
}

variable "c" {
  type = number
  deafult = 123
}

output "a" {
  value = var.a
}

output "b" {
  value = var.b
}

output "c" {
  value = var.c
}
```
