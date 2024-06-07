# learning-terraform

- Terraform is a IaaC solution. It works for almost every cloud
  provider we can think of. Unlike, AWS cloudformation which is AWS
  specific and uses JSON for defining the template, Terraform can be
  used for creating infrastructure/resources on various cloud providers and uses a more readable language (Hashicorp)

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
terraform apply
terraform plan
```
