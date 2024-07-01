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
