---
title: "Dataflow: Amazon Web Services"
date: "2023-08-15"
draft: false
tags: ["Dataflow", "Data Lake", "Batch Ingestion", "Data Sink", "ETL", "AWS"]
author: "Jason Grein"
description: "Evaluating Batch Data Ingestion Pipelines with Data Simulator logs to AWS"

cover:
  image: "cover-photo.png"
#  alt: "AI Generated Superhero Photo"
#  caption: "These guys are ready to duke it out for Data!"
#  relative: false # To use relative path for cover image, used in hugo Page-bundles

showToc: true
TocOpen: false
hidemeta: false
comments: true

disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

Here's the first post in a series of Cloud Platform comparison evaluations on Data Simulator batch ingestion pipelines and ETL jobs. In this post, we're interested in developing a dataflow pipeline which will sink log data to AWS S3 blob storage buckets, and transform the log data batches from JSON to Parquet. This is a particularly useful Data Ops practice to scale data pipelines across multiple Cloud Providers which can inherit more resilient uptime and access to data while reducing vendor lock-in or lacking capabilities.

**Motivation**: Build a Data Ingestion Pipeline to AWS S3 and a Serverless ETL Function with Lambda to standardize Data Simulator log data from JSON to Parquet for the purpose of creating a benchmark when comparing AWS against other Cloud Platforms. This will help to measure costs & benefits with consideration to AWS in terms of reliable operations delivery, efficient IaaC development, and cost optimization.

## Prerequisites

### Pub/Sub & Message Queue

This exercise will leverage the work done through the [Pub/Sub & Message Queue Series](/pub-sub/comparison) which delivered simulation data through some popular Sub/Sub & MQ services. In particular, we'll be leveraging Kafka to integrate the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) logs to our cloud storage layers. Please refer back to [This Post](/pub-sub/kafka#objective) to familiarize yourself with how to operate Kafka.

### AWS Account

If you don't have an account registered with AWS, then you may sign-up [Here](https://portal.aws.amazon.com/gp/aws/developer/registration/index.html). An AWS account will be required to spin-up the infrastructure required for this demonstration, and to proceed with the following configuration steps:

{{< details header="Video Demonstration" icon="icons/video-icon.svg" >}}
  {{< google-drive-video "1I2CMXyav0zgGjhgQM_ivtdOZGba8sQUN" >}}
{{< /details >}}

#### Instructions
{{< details header="Create New User" markdown="true" >}}
  * Create a new Policy with [IP Restrictions](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_aws_deny-ip.html) so only users in our security group can interact with AWS. This is helpful incase our security tokens are compromised, so even with unauthorized use malicuous actors still wont be able to access our AWS account on another IP address

  ```
  {
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Deny",
        "Action": "*",
        "Resource": "*",
        "Condition": {
            "NotIpAddress": {
                "aws:SourceIp": [
                    "192.0.2.0/24",
                    "INSERT IP HERE"
                ]
            },
            "Bool": {"aws:ViaAWSService": "false"}
        }
    }
  }
  ```

  * Create a [New IAM Group](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups_create.html) which we can bind the IP Whitelist policy above to

  * Create a [New IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) **without** access to AWS Management Console - this user should only be able to access AWS through CLI.
    * [Add User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console) to the the new IAM Group to bind the IP Whitelist policy
    * Add the `IAMFullAccess` role to this user's [Permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_change-permissions.html#users_change_permissions-add-console)
    * Create a [CLI Access Credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey), and don't leave page until the Shared Credentials have been completed in the following steps
{{</ details >}}

{{< details header="Install & Configure CLI" markdown="true" >}}
 * Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
 * Configure [Shared Credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) with new credentials created in the earlier step
 ```bash
 aws configure --profile=datasim-example
 ```
{{</ details >}}

{{< details header="Configure Terraform Backend" markdown="true" >}}
 * Create a Private [S3 Bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-bucket.html) for Terraform `tfstate`. This will reduce the chance of sensitive data being committed to source code.
 * Create a new [DynamoDB Table](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/getting-started-step-1.html) in AWS with a Partition Key named [LockID](https://developer.hashicorp.com/terraform/language/settings/backends/s3#dynamodb-state-locking)
 * [Configure Terraform Backend](https://developer.hashicorp.com/terraform/language/settings/backends/s3) to store `tfstate` to S3
   * Create a new policy for user access to these new resources
   ```
   {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::INSERT_BUCKET_NAME"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::INSERT_BUCKET_NAME/INSERT_PATH_KEY"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeTable",
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/INSERT_TABLE_NAME"
        }
     ]
   }
   ```

   * Add configurations to the `batch/serverless_functions/aws/backend.conf` terraform file
   ```
   bucket         = "INSERT_BUCKET_NAME" #example: datasim-example-terraform-tfstate-bucket
   key            = "INSERT_PATH_KEY"    #example: dev/terraform/state
   region         = "us-east-1"
   encrypt        = true
   dynamodb_table = "INSERT_TABLE_NAME"  #example: datasim-example-terraform-tfstate-table
   profile        = "datasim-example"
   ```
{{</ details >}}

{{< details header="Configure Terraform tfvars" markdown="true" >}}
  Configure the following into the terraform input file `batch/serverless_functions/aws/terraform.tfvars`
  ```
  aws_region               = "us-east-1"
  new_service_account_name = "datasim-example-dataflow"
  contributor_user         = "datasim-example-user" #Created in Step 1
  aws_cli_profile          = "datasim-example"
  ```
{{</ details >}}

{{< details header="Optional" markdown="true" >}}
 * Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) for local debugging
 * Install [AWS Toolkit for Visual Studio Code](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/setup-toolkit.html) for creating function templates and accelerating build and debug purposes
{{</ details >}}

## Install

Clone this repository to local computer **[py-superhero-dataflow](https://github.com/jg-ghub/py-superhero-dataflow)** {{< svg-icon "github-icon" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-dataflow
```

## Services
{{< details header="Schematic" icon="icons/flow-icon.svg" >}}
  {{< get-html "content/data-flow/batch/aws/worker-schematic.html" >}}
{{</ details >}}

### Batch Data Ingestion Worker

Similar to the [Original Kafka Worker](/pub-sub/kafka#worker), this worker is consuming messages from Kafka Topics to clean up stale records in Redis, and also persist the log messages to a S3 Blob Storage account in AWS called `raw`. This new worker watches a specific Kafka Topic like `lib.server.lobby`, and is triggered each time it receives a new message. However, unlike the Original Worker which passes individual log message as JSON to the data sink, this Worker will poll the Topic until a desired amount of un-processed messages have been established to pass log messages as JSON in bulk - desired amount configurable with the `BATCH_SIZE` ENV variable.

More details on the ingestion functionality can be found in the [Batch Ingestion Worker](/data-flow/batch/comparison#kafka-batch) article.

#### Local Deployment
To test/debug the Batch Data Ingestion service, we can start up the services required to begin generating data for consumption by executing the following locally in bash.

```bash
export $(cat batch/kafka/env/kafka.env) && \
export $(cat batch/kafka/env/local.env) && \
docker compose --env-file kafka/.env -f kafka/compose/docker-compose.kafka.yml up -d zookeeper kafka kafka-ui
#Wait for Kafka-ui to be up and connected to Kafka cluster
export PLAYERS=10 && \
docker compose -f batch/kafka/compose/docker-compose.local.yml up -d redis build_db superhero_server superhero_nginx player --scale player=${PLAYERS}
```

Then we can begin deploying worker services locally for debugging with

```bash
export $(cat batch/kafka/env/kafka.env) && \
export $(cat batch/kafka/env/local.env) && \
export WORKER_CHANNEL=lib.server.lobby && \
python3 batch/kafka/worker.py
```

### Serverless Function ETL Worker

A serverless function is code that can be executed in a cloud environment without the developer needing to manage the underlying server infrastructure. Serverless computing is a cloud computing paradigm where cloud providers automatically manage the allocation of resources and scaling of applications, allowing developers to focus solely on writing and deploying code. Our Serverless Function in this demonstration utilizes [AWS Lambda](https://aws.amazon.com/lambda/) with a [S3 Blob Trigger](https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html) configuration to execute our ETL job with each JSON log file delivered to our *raw* data sink. The ETL job is fairly simple, it's 
 * **Extract** JSON log files from the S3 bucket *raw* data sink
 * **Transform** the log data to a columnar data structure with respect to the log model (`lib.server.lobby`, `'lib.server.game`, `log.meta`, etc)
 * Finally **Load** the data to Parquet in the *standard* S3 bucket data sink.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fsource%2Flambda_function.py%23L83-129&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

The unique aspect to this ETL worker within the AWS Platform can be seen within the Extract and load functions. Specifically, we're using [`boto3`]("https://boto3.amazonaws.com/v1/documentation/api/latest/index.html") to not only reads/writes the data to S3, but it's also providing us [Session Credentials](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#boto3.session.Session.get_credentials) to authenticate with S3 private buckets from within Lambda. This helps to secure the process and adhere to security protocols while eliminating the need to store sensitive credentials close to the source.

#### Local Testing

Replicating the Lambda Function invocation locally is somewhat possible by mocking a HTTP Lambda function with a blob trigger payload to evaluate how the code can be triggered. We can utilize [AWS Serverless Application Model (SAM) CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) and the [AWS Toolkit for Visual Studio Code](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/setup-toolkit.html) to simulate a Lambda function hosted locally on our machine, but we won't dive very deep into this scenario as we consider it out-of-scope for this demonstration. More detail can be found in this [AWS Blog Post](https://aws.amazon.com/blogs/developer/announcing-aws-toolkit-for-visual-studio-code/) on how to start testing Lambda locally.

## Objective

In this demonstration, we want to evaluate exactly how much infrastructure is required to spin up a Batch Data Ingestion Sink, and an ETL pipeline to standardize JSON log data into a Parquet data structures suitable for Data Warehousing and Advance Analytics in the future. The goal is to be able to benchmark both qualitatively and quantitatively how AWS stacks up against other cloud platforms like Azure and GCP. We will leverage the [Pub/Sub & Message Queue](#pubsub--message-queue) mentioned earlier to ingest data into AWS for our demonstration.

### Infrastructure

The contents of the IaaC for this demonstration is structured with Terraform Modules under the style guide from [Hashicorp Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure). Each component of this deployment is a nested module with custom configurations tailored for our solution.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmain.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Service Account Auth

The Service Account Auth Module will authenticate with AWS using the IP Bound Service Account to request Access Tokens, and
create new least-privileged Service Accounts to Deploy/Destroy our Data Simulator Dataflow. The idea here is that the user authorized with the AWS CLI has at least the `IAMFullAccess` roll to create Service Accounts with up-to unlimited access in S3 (although discouraged), but this curator is restricted access to AWS based on the IP restriction we setup in the [Prerequisites](#aws-account).

This is a useful way to keep projects in check, and only provide the minimum role-sets required to reduce any potential harm in the case where a Service Account is compromised. However, in this demonstration the Lambda Function isn't deployable without full role-set privilege which breaks our specially crafted design to reduce Service Account access.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmodules%2Fservice_account_auth%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Buckets

Very simple module to create private S3 Storage Buckets.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmodules%2Fsuperhero_buckets%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Security Groups

Create a security group with access rights to the *raw* S3 Storage Bucket, and assign membership to the user account responsible for the Batch Data Ingestion with this group. Creating security groups brings efficiency and organization to resource access by defining groups with specific business purposes and limited access to relevant resources. This could include grouping users by business function (like engineer, analyst), or by geographical location (North America vs Europe), etc. However, it's important to distinguish draw backs from assigning members to security groups in Terraform - membership needs to be assigned completely in Terraform or in the Portal. If members were to be assigned originally by Terraform, and then later added by portal, then the next deployment won't take into consideration those members added after-the-fact via the portal, and will eventually lose privileges/access.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmodules%2Fsuperhero_security_groups%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Function Layers

This module is specific to AWS Lambda, which requires [Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/chapter-layers.html) be created for any additional libraries required by the function. This is to help keep Lambda Deployment packages small, and dependency libraries re-usable instead of duplicating the dependency across similar Lambda Function Deployment packages. From my experience across the different platforms, I believe the Lambda Layers is an added complexity which takes away from a point drawn from their developer guide: *"promotes separation of concerns and helps you focus on your function logic"*. If anything, you spend more time trying to fix breaks caused by missing dependencies than pushing core functionality...

Here in this module, we install independent python dependency libraries into target folders to then zip them as a Lambda Layer. Lambda Layers have a small storage size and can be easily broken by adding too many dependency libraries to a single layer. This is why in this example, we install both `numpy` and `pyarrow` as independent zip files, stored in the *functions* S3 Storage Bucket, and then finally loaded as a Lambda Layer.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmodules%2Fsuperhero_functions_layers%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Function

The core component to this demonstration, deploying and configuring the Lambda Function. This module helps to automate zipping the deployment source and uploading to the *functions* S3 Storage Bucket, which is then incorporated into the Lambda Function Deployment. It's also responsible for some important configurations like
 * Attaching the Lambda Layers to include deployment package library dependencies
 * Authorizing the Lambda Function Resource to S3 buckets like *standard* for the ETL job
 * Configuring the Blob Storage Trigger when JSON logs are stored in the *raw* S3 Storage Bucket
 * Initializing Cloud Watch to monitor Lambda Function invocations and log function execution and errors, which isn't configured by default
 * Miscellaneous IAM policies to make sure the Function is working as it should

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Faws%2Fmodules%2Fsuperhero_functions%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

### Production Deployment

{{< details header="Video Demonstration" icon="icons/video-icon.svg" >}}
  {{< google-drive-video "1xfgG7nHJab9w99ExkbzT5aL0N3DkHpni" >}}
{{< /details >}}

Spinning up this Data flow with Batch Data Ingestion to AWS S3 and a Lambda Function ETL Job to standardize our Data Simulator Log data to Parquet can be accomplished with the following deployment commands

```bash
terraform -chdir=batch/serverless_functions/aws init -backend-config=$(pwd)/batch/serverless_functions/aws/backend.conf &&
terraform -chdir=batch/serverless_functions/aws plan &&
terraform -chdir=batch/serverless_functions/aws apply -auto-approve
```

```bash
export $(cat batch/kafka/env/kafka.env) && \
export OUTPUT_DIR=$(terraform -chdir=batch/serverless_functions/aws output -raw raw_bucket_id)
export AWS_PROFILE=$(terraform -chdir=batch/serverless_functions/aws output -raw aws_cli_profile)
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.aws.yml build &&
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.aws.yml up -d zookeeper kafka kafka-ui

#Wait for Kafka-ui to be up and connected to Kafka cluster
export PLAYERS=10
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.aws.yml up --scale player=${PLAYERS}
```

In this demonstration, we've started to think about deploying data architecture in the cloud to build a scalable infrastructure for Data Ops. We chose AWS as our cloud platform to store our data in S3 Blob Storage data sinks, and standardize the log data in Parquet format using an ETL Job within AWS Lambda Serverless Functions. The Data Ingestion was of course operating locally with our Data Simulator application batch processing kafka messages into JSON logs delivered to AWS S3 Buckets. We also made some proactive guard rails around our infrastructure deployment pipeline by whitelisting our IP against the new project user which will greatly reduce unforeseen impacts to our AWS Account in the case our user is compromised. Finally, we were able to establish some Data Governance requirements by creating Security Groups in AWS and allowing resource access to select S3 Buckets - restricting accessibility to data to users/resources with specific use cases, which in our case was staging data from JSON to Parquet.

**Qualitatively** my opinion in regard to how AWS performed in this exercise was very good with some complications. AWS of course is a well established product which was first-to-market of all the major cloud competitors, and likely has the widest range of available features to offer its customers. Finding the necessary resources for this project wasn't very complicated, and there was a lot of literature on the topics already to accelerate the infrastructure development. However, I felt there were a few efficiency roadblocks to getting this project fully functional.

 * **IAM policies** AWS IAM Policies are incredibly robust and have almost limitless configuration possibilities to ensure you're getting the highest levels of Security for your project. Unfortunately, this level of granularity can be a struggle to grasp quickly and keep straight between the different types of policies like user policies vs resource policies, etc. In my experience, it took a while to get comfortable writing IAM policies, and feel like other platforms have simplified this security parameter to make engineering the deployment faster with less hurdles.

 * **Lambda Layers** Lambda Serverless Function dependencies was probably the biggest hurdle in this experiment. Having to separately bundle the core functionality of the function from the dependency required a couple different complexities which took away from the speed of development. Installing dependencies for Lambda Layers requires specific package management instructions which are somewhat buried in documentation, and having limited space in a Lambda Layer requires that big dependencies be installed/zipped into their own layer requiring more trial and error attempts.

**Quantitatively** our Data Ingestion Pipeline and Serverless ETL Function was a success. The Data Ops were satisfied with this build, and running the experiment costed us no money during the trials. Deployment time is fairly quick taking ~2 minutes to deploy and ~30 seconds to tear down. Following the [Don't Repeat Yourself (DRY) Guidelines](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/), we were able to build this deployment pipeline with 5 distinct types of modules for a total of ~500+ lines of Terraform code.

AWS is a fantastic platform to work with, and easily meets all of our expectations to get this demonstration up and running. We hope if you ever intend to work with AWS, or are currently working with AWS and are trying to build a similar capability, that you accelerate your development by using this template we've built. Also, in case of just starting out with AWS, please consider following the security protocols within this package to reduce any unforeseen consequences of account compromise. Too many poorly written how-to guides are floating around popular article sites which can easily be compromised and rack up incredible amounts of debt on these cloud platforms.