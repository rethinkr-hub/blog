---
title: "Dataflow: Google Cloud Platform"
date: "2023-08-28"
draft: false
tags: ["Dataflow", "Data Lake", "Batch Ingestion", "Data Sink", "ETL", "GCP"]
author: "Jason Grein"
description: "Evaluating Batch Data Ingestion Pipelines with Data Simulator logs to GCP"

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

In this last post assessing Cloud Platforms focused on Data Simulator batch ingestion pipelines and ETL jobs, we focus on creating a dataflow pipeline in Google Cloud Platform (GCP). This pipeline will be responsible for transferring log data to GCP Cloud Storage Buckets and converting the log data batches from JSON to Parquet format. This is a particularly useful Data Ops practice to scale data pipelines across multiple Cloud Providers which can inherit more resilient uptime and access to data while reducing vendor lock-in or lacking capabilities.

**Motivation**: Build a Data Ingestion Pipeline to Google Cloud Storage (GCS) Buckets and a Serverless ETL Function with GCP Cloud Functions to standardize Data Simulator log data from JSON to Parquet for the purpose of creating a benchmark when comparing GCP against other Cloud Platforms. This will help to measure costs & benefits with consideration to GCP in terms of reliable operations delivery, efficient IaaC development, and cost optimization.

## Prerequisites

### Pub/Sub & Message Queue

This exercise will leverage the work done through the [Pub/Sub & Message Queue Series](/pub-sub/comparison) which delivered simulation data through some popular Sub/Sub & MQ services. In particular, we'll be leveraging Kafka to integrate the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) logs to our cloud storage layers. Please refer back to [This Post](/pub-sub/kafka#objective) to familiarize yourself with how to operate Kafka.

### GCP Account

If you don't have an account registered with GCP, then you can sign-up [Here](https://console.cloud.google.com/freetrial). A GCP project will be required to spin-up the infrastructure required for this demonstration, and to proceed with the following configuration steps:

{{< details header="Video Demonstration" icon="icons/video-icon.svg" >}}
  {{< google-drive-video "1_DHu6H4BVYPuo4-Hfu1T2DRED8j3mwxo" >}}
{{< /details >}}

#### Instructions
{{< details header="Create New Project" markdown="true" >}}
  * Create a [New GCP Project](https://developers.google.com/workspace/guides/create-project)
  * Create a [New Gmail Account](https://support.google.com/mail/answer/56256?hl=en)
  * Grant this new Gmail Account [User Access](https://cloud.google.com/iam/docs/granting-changing-revoking-access#iam-view-access-console) with limited privileges
    * `Viewer Role`
    * `Storage Admin Role`
    * `Service Usage Consumer Role`
  * Create a [New Service Account](https://cloud.google.com/iam/docs/service-accounts-create) who is consumer/used by the New Gmail Account, and administered by the Product Owner Account. Provide permissions with administrative roles
    * `Security Admin Role`
    * `Service Account Admin Role`
  * Bind the `Account Token Generator Role` in the Servie Account permissions to the New Gmail Account  
{{</ details >}}

{{< details header="Enable APIs" markdown="true" >}}
  * Enable [Cloud Resource Manager API](https://console.developers.google.com/apis/library/cloudresourcemanager.googleapis.com)
  * Enable [Cloud Functions API](https://console.developers.google.com/apis/library/cloudfunctions.googleapis.com)
  * Enable [Cloud Build API](https://console.developers.google.com/apis/library/cloudbuild.googleapis.com)
  * Enable [Identity And Access Management (IAM) API](https://console.cloud.google.com/apis/library/iam.googleapis.com)
{{</ details >}}

{{< details header="Configure CLI and Terraform" markdown="true" >}}
  * Install [gcloud CLI](https://cloud.google.com/sdk/docs/install)
  * Configure [Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials)
  ```bash
  gcloud config set project INSERT_PROJECT_ID_HERE
  gcloud auth login
  gcloud auth application-default login

  #Verify Auth User
  gcloud auth list
  ```

  * Configure Terraform `batch/serverless_functions/gcp/terraform.tfvars` file
  ```
  project_id                        = "INSERT_PROJECT_ID_HERE"
  impersonate_service_account_email = "INSERT_SERVICE_ACCOUNT_EMAIL_HERE"
  impersonate_user_email            = "INSERT_NEW_GMAIL_ACCOUNT_EMAIL_HERE"
  ```
{{</ details >}}
  
{{< details header="Configure Terraform Backend" markdown="true" >}}
  * Create a GCS Bucket for Terraform tfstate files, so this is not stored locally and risks getting pushed to source repository. Update this bucket name in the backend.conf file
    * Creating a [GCS Bucket](https://cloud.google.com/storage/docs/creating-buckets)
    * Configuring [Terraform tfstate Backend to GCS](https://cloud.google.com/docs/terraform/resource-management/store-state) to `batch/serverless_functions/gcp/backend.conf`
    ```
    bucket = "INSERT_BUCKET_NAME_HERE"
    prefix = "prod/terraform/state"
    ```
{{</ details >}}

{{< details header="Optional" markdown="true" >}}
  * [Optional] Install the [Google Cloud Code](https://cloud.google.com/code) extension for Visual Studio Code
{{</ details >}}

## Install

Clone this repository to local computer **[py-superhero-dataflow](https://github.com/jg-ghub/py-superhero-dataflow)** {{< svg-icon "github-icon" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-dataflow
```

## Services
{{< details header="Schematic" icon="icons/flow-icon.svg" >}}
  {{< get-html "content/data-flow/batch/gcp/worker-schematic.html" >}}
{{</ details >}}

### Batch Data Ingestion Worker

Similar to the [Original Kafka Worker](/pub-sub/kafka#worker), this worker is consuming messages from Kafka Topics to clean up stale records in Redis, and also persist the log messages to a GCS Bucket in GCP called `raw`. This new worker watches a specific Kafka Topic like `lib.server.lobby`, and is triggered each time it receives a new message. However, unlike the Original Worker which passes individual log messages as JSON to the data sink, this Worker will poll the Topic until a desired amount of un-processed messages have been established to pass log messages as JSON in bulk - desired amount configurable with the `BATCH_SIZE` ENV variable.

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

A serverless function is code that can be executed in a cloud environment without the developer needing to manage the underlying server infrastructure. Serverless computing is a cloud computing paradigm where cloud providers automatically manage the allocation of resources and scaling of applications, allowing developers to focus solely on writing and deploying code. Our Serverless Function in this demonstration utilizes [Cloud Functions](https://cloud.google.com/functions) with a [Cloud Storage Trigger](https://cloud.google.com/functions/docs/calling/storage) configuration to execute our ETL job with each JSON log file delivered to our *raw* data sink. The ETL job is fairly simple, it's 
 * **Extract** JSON log files from the GCS Bucket *raw* data sink
 * **Transform** the log data to a columnar data structure with respect to the log model (`lib.server.lobby`, `'lib.server.game`, `log.meta`, etc)
 * Finally **Load** the data to Parquet in the *standard* GCS Bucket data sink.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fgcp%2Fsource%2Fmain.py%23L81-121&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

The unique aspect to this ETL worker within the Azure Platform can be seen within the Extract and load functions. Specifically, we're using [`gcsfs`](https://gcsfs.readthedocs.io/en/latest/) to not only reads/writes the data to GCS, but it's also handling [Authentication](https://gcsfs.readthedocs.io/en/latest/#credentials) for us automatically to pass credentials with GCP from within Google Cloud Functions. This helps to secure the process and adhere to security protocols while eliminating the need to store sensitive credentials close to the source.

#### Local Testing

Google Cloud Functions can be [Developed Locally](https://cloud.google.com/functions/docs/running/overview) and run locally with two possible abstraction layers. [Functions Framework](https://cloud.google.com/functions/docs/functions-framework) allows you to spin up a development server effortlessly to mimic calling Cloud Function on your local workstation. The Functions Framework can be tested not only with HTTP requests, but can also be used to mimic other Cloud Events such as a Blob Trigger event to evaluate a New Blob Creation event body. [Functions Emulator](https://cloud.google.com/functions/docs/running/functions-emulator) is a new alpha release local development tool which embodies the function in a Docker Container which can run/test the function in a local environment which closer resembles a production deployment.

## Objective

In this demonstration, we want to evaluate exactly how much infrastructure is required to spin up a Batch Data Ingestion Sink, and an ETL pipeline to standardize JSON log data into a Parquet data structures suitable for Data Warehousing and Advance Analytics in the future. The goal is to be able to benchmark both qualitatively and quantitatively how GCP stacks up against other cloud platforms like AWS and Azure. We will leverage the [Pub/Sub & Message Queue](#pubsub--message-queue) mentioned earlier to ingest data into Azure for our demonstration.

### Infrastructure

The contents of the IaaC for this demonstration is structured with Terraform Modules under the style guide from [Hashicorp Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure). Each component of this deployment is a nested module with custom configurations tailored for our solution.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fgcp%2Fmain.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Service Account Auth

The Service Account Auth module utilizes GCP's [Service Account Impersonation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) to generate short lived access tokens with an authenticated Service Principal bound with the `roles/iam.serviceAccountTokenCreator` IAM role. Now this Service Account Impersonator can create new Service Accounts with least-privileged roles to Deploy/Destroy our Data Simulator Dataflow. The idea here is that there is a GCP Account in the Project which is bound to a Service Principal with the [Service Account Token Creator](https://cloud.google.com/iam/docs/understanding-roles#iam.serviceAccountTokenCreator) role. This Service Principal can only be access through a specific GCP Account, and can now create new Service Principals with bootstrapped roles for specific business cases.

This is a useful way to keep projects in check, and only provide the minimum role-sets required to reduce any potential harm in the case where a Service Account is compromised.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fgcp%2Fmodules%2Fservice_account_auth%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Buckets

Very simple module to create private GCS Buckets.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fgcp%2Fmodules%2Fsuperhero_buckets%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Function

Here the Function module will zip the contents of the core function with a list of package dependencies in the `requirements.txt` file, and upload the zipped function to the GCS *functions* bucket. Next, we'll spin up a Cloud Function with the function linked to our zipped package in the *functions* bucket, and configured with the Blog Trigger event type to execute on each object passed to the *raw* GSC Bucket. Finally, we'll bind the `roles/cloudfunctions.invoker` IAM role to the Service Principal which created the Cloud Function to allow Blob Trigger invocations.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fgcp%2Fmodules%2Fsuperhero_functions%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

### Production Deployment

Spinning up this Data flow with Batch Data Ingestion to GCS Buckets and a Google Cloud Function ETL Job to standardize our Data Simulator Log data to Parquet can be accomplished with the following deployment commands

{{< details header="Video Demonstration" icon="icons/video-icon.svg" >}}
  {{< google-drive-video "1GABZbfenvvu_E3YaZmm3jPI-Ans0hbDG" >}}
{{< /details >}}

```bash
terraform -chdir=batch/serverless_functions/gcp init -backend-config=$(pwd)/batch/serverless_functions/gcp/backend.conf &&
terraform -chdir=batch/serverless_functions/gcp plan &&
terraform -chdir=batch/serverless_functions/gcp apply -auto-approve
```

```bash
export $(cat batch/kafka/env/kafka.env) && \
export GOOGLE_CLOUD_PROJECT=$(terraform -chdir=batch/serverless_functions/gcp output -raw project_id) && \
export OUTPUT_DIR=$(terraform -chdir=batch/serverless_functions/gcp output -raw raw_bucket_name) && \
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.gcp.yml build &&
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.gcp.yml up -d zookeeper kafka kafka-ui

#Wait for Kafka-ui to be up and connected to Kafka cluster
export PLAYERS=10
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.gcp.yml up --scale player=${PLAYERS}
```

In this demonstration, we've implemented a data architecture within the cloud environment to construct a scalable foundation for Data Ops. Our chosen cloud platform was GCP, where we utilized GCS Buckets for persisting data. Additionally, we employed an ETL Job within GCP Cloud Functions to format log data into Parquet from the raw Data Lake layer to the standardized layer. The process of Data Ingestion was run locally by our Data Simulator application. This application conducted batch processing on Kafka messages, resulting in JSON logs that were directed to GCS Buckets.

We unfortunately fall short on some Data Governance requirements with the lack of Security Groups in GCP - [Creating Groups](https://support.google.com/a/answer/9400082?sjid=6375052622175110603-NA#step1&zippy=%2Cstep-create-a-group) requires a Google Organization which extends past free tier usage. Therefore, consciously restricting data accessibility to users and resources needs to be completed by user instead of groups with specific use cases. This makes adhering to Data Governance more granular and difficult to maintain. However, we still managed to maintain some proactive guard rails around our infrastructure deployment pipeline by invoking the Service Account Impersonation steps to allow only a single privileged user the ability to create new Service Accounts which reduces the attack surface of our GCP Project getting compromised.

**Qualitatively** my opinion in regard to how GCP performed in this exercise was great with a single letdown. GCP is a newer player to cloud offerings compared to AWS, but feels easier to use with this limited use case. Finding the necessary resources for this project was very simple, and there was a lot of literature on the topics already to accelerate the infrastructure development. The only drawback was the inability to create Security Groups

 * **Security Groups** allow us to organize role sets based on business specific use cases, and then assign users to the group. This reduces the amount of repetitive work assigning all roles and access to users by simply assigning them to a predefined group. Security Groups can be created from Google Groups, but Google Groups can only be created Organization Admins which isn't offered in GCP. Instead, this seems to be a capability closer to Google Identity which sits closer to G Suit.

**Quantitatively** our Data Ingestion Pipeline and Serverless ETL Function was a success. The Data Ops were satisfied with this build, and running the experiment costed us no money during the trials. Deployment time is fairly quick taking ~2 minutes to deploy and ~30 seconds to tear down. Following the [Don't Repeat Yourself (DRY) Guidelines](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/), we were able to build this deployment pipeline with 3 distinct types of modules for a total of ~300+ lines of Terraform code. However, with the reduction in code length and modules comes less Data Governance efficiency and sustainability around Security Groups.

GCP is a fantastic platform to work with, and easily meets all of our expectations to get this demonstration up and running. We hope if you ever intend to work with GCP, or are currently working with GCP and are trying to build a similar capability, that you accelerate your development by using this template we've built. Also, in case of just starting out with GCP, please consider following the security protocols within this package to reduce any unforeseen vulnerabilities around persisting security keys close to the source code. Too many poorly written how-to guides are floating around popular article sites which can easily be compromised and rack up incredible amounts of debt on these cloud platforms.