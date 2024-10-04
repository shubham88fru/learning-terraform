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

```terraform
//List type
variable "a" {
  type = list(string)
  default = ["foo", "bar", "baz"]
}

output "a" {
  value = var.a
}

output "b" {
  value = element(var.a, 1) //return item at index 1 in the list.
}

output "c" {
  value = length(var.a)
}
```

```terraform
//Set type
variable "my_set" {
  type = set(number)
  deafult = [7, 2, 2]
}

variable "my_list" {
  type = list(string)
  deafult = ["foo", "bar", "foo"]
}

output "set" {
  value = var.my_set
}

output "list" {
  value = var.my_list
}

output "list_as_set" {
  value = toset(var.my_list)
}
```

```terraform
//Tuple type
variable "my_tup" {
  type = tuple([number, string, bool])
  default = [4, "hello", false]
}

output "tup" {
  value = var.my_tup
}
```

```terraform
//Map type
variable "my_map" {
  type = map(number) //a where values will be numbers.
  default = {
    "alpha" = 2
    "bravo" = 3
  }
}

output "map" {
  value = var.my_map
}

output "alpha_value" {
  value = var.my_map["alpha"]
}
```

```terraform
//Object type
variable "person" {
  type = object({ name = string, age = number })
  default = {
    name = "bob"
    age = 10
  }
}

output "person" {
  value = var.person
}

variable "person_with_address" {
  type = object({ name=string, age=number, address=object({
    line1=string, line2=string, county=string, postcode=string})
    })

  default = {
    name = "Jim"
    age = 21
    address = {
      line1 = "1 the road"
      line2 = "St Ives"
      county = "Cambridgeshire"
      postcode = "CB1 2GB"
    }
  }
}

output "person_with_address" {
  value = var.person_with_address
}
```

```terraform
//Any type - The any type is a special constrcut that
//serves as a placeholder for a type yet to be decided. `any` itself
//is not a type. Terraform will attempt to calculate the type at
//runtime when you use `any`.
variable "any_example" {
  type = any
  default = {
    field1 = "foo"
    field2 = "bar"
  }
}

output "any_example" {
  value = var.any_example
}
```

```terraform
//modules.
//A module in terraform is a mini Terraform project in itself
//that can contain all of the same constructs as our main
//Terraform project(resources, data blocks, locals, etc.)
//Modules let us define a reusable block of Terraform code.

/*
  Folder structure.
  - sqs-with-backoff/
      -- main.tf
      -- variables.tf
      -- output.tf
  - main.tf
*/

// `/main.tf`
provider "aws" {
  region = "eu-west-1"
}

module "work_queue" {
  source = "./sqs-with-backoff" //define the folder as a moudle.
  queue_name = "work-queue"
}

output "work_queue_name" {
  value = module.work_queue.queue_name
}

output "work_queue_dead_letter_name" {
  value = module.work_queue.dead_letter_queue_name
}

// `/sqs-with-backoff/variables.tf`
//Variables have a special meaning when used
//with a module; they become the input values for
//your moudle.
//If a default value is not provided for variable
//in a module, it will have to be provided when using
//the module.
variable "queue_name" {
  description = "Name of queue"
}

variable "max_receive_count" {
  description = "The maximum number of times that a message can be received by consumers"
  default = 5
}

variable "visibility_timeout" {
  deafult = 30
}

// `/sqs-with-backoff/output.tf`
output "queue_arn" {
  value = aws_sqs_queue.sqs.arn
}

output "queue_name" {
  value = aws_sqs_queue.sqs.name
}

output "dead_letter_queue_arn" {
  value = aws_sqs_queue.sqs_dead_letter.arn
}

output "dead_letter_queue_name" {
  value = aws_sqs_queue.sqs_dead_letter.name
}

// `/sqs-with-backoff/main.tf`
resource "aws_sqs_queue" "sqs" {
  name = "awesome_co-${var.queue_name}"
  visibility_timeout_seconds = var.visibility_timeout
  delay_seconds = 0
  max_message_size = 262144
  message_retention_seconds = 345600
  receive_wait_time_seconds = 20
  redrive_policy= "{\"deadLetterTargetArn\":\"${aws_sqs_queue.sqs_dead_letter.arn}\", \"maxReceiveCount\": ${var.max_receive_count}}"
}

resource "aws_sqs_queue" "sqs_dead_letter" {
  name = "awsome_co_${var.queue_name}-dead-letter"
  delay_seconds = 0
  max_message_size = 262144
  message_retention_seconds = 1209600
  receive_wait_time_seconds = 20
}
```
