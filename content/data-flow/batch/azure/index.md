---
title: "Dataflow: Azure"
date: "2023-08-28"
draft: true
tags: ["Dataflow", "Data Lake", "Batch Ingestion", "Data Sink", "ETL", "Azure"]
author: "Jason Grein"
description: "Evaluating Batch Data Ingestion Pipelines with Data Simulator logs to Azure"

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

The next entry in a sequence of assessments comparing Cloud Platforms focuses on Data Simulator batch ingestion pipelines and ETL jobs. In this installment, our focus is on creating a dataflow pipeline. This pipeline will be responsible for transferring log data to Azure ADLS Gen2 Storage Accounts and converting the log data batches from JSON to Parquet format. This is a particularly useful Data Ops practice to scale data pipelines across multiple Cloud Providers which can inherit more resilient uptime and access to data while reducing vendor lock-in or lacking capabilities.

**Motivation**: Build a Data Ingestion Pipeline to Azure Storage Accounts and a Serverless ETL Function with Azure Functions to standardize Data Simulator log data from JSON to Parquet for the purpose of creating a benchmark when comparing Azure against other Cloud Platforms. This will help to measure costs & benefits with consideration to Azure in terms of reliable operations delivery, efficient IaaC development, and cost optimization.

## Prerequisites

### Pub/Sub & Message Queue

This exercise will leverage the work done through the [Pub/Sub & Message Queue Series](/pub-sub/comparison) which delivered simulation data through some popular Sub/Sub & MQ services. In particular, we'll be leveraging Kafka to integrate the [Super Hero Combat Data Simulator](/data-simulators/superhero-combat) logs to our cloud storage layers. Please refer back to [This Post](/pub-sub/kafka#objective) to familiarize yourself with how to operate Kafka.

### Azure Account

If you don't have an account registered with Azure, then you sign-up [Here](https://azure.microsoft.com/en-ca/free/). An Azure account will be required to spin-up the infrastructure required for this demonstration, and to proceed with the following configuration steps:

#### Instructions
{{< details header="Configure Azure and Terraform Variables" markdown="true" >}}
 * Create a [New Subscription](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/create-subscription)
 * Create a [New Resource Group](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal)
 * Create a [New AD User](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/add-users) and provide the following permissions (take not of [UPN](https://learn.microsoft.com/en-us/azure/active-directory/hybrid/connect/plan-connect-userprincipalname))
   * `Storage Blob Data Owner`
   * `Contributor`
 * Add the following details to the `batch/serverless_functions/aws/terraform.tfvars` file
 ```
 subscription_id = "INSERT_SUBSCRIPTION_ID_HERE"
 tenant_id       = "INSERT_TENANT_ID_HERE"
 contributor_upn = "INSERT_NEW_AD_USER_UPN_HERE"
 ```
{{</ details >}}

{{< details header="Install CLI" markdown="true" >}}
 * Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
 * Login to Azure with the CLI
 ```bash
 az login
 ```
{{</ details >}}

{{< details header="Configure Terraform Backend" markdown="true" >}}
 * Create an ADLS Gen2 Storage Account for Terraform tfstate files, so this is not stored locally and risks getting pushed to source repository. Update this bucket name in the backend.conf file
   * Creating a [ADLS Gen2 Storage Account](https://learn.microsoft.com/en-us/azure/storage/blobs/create-data-lake-storage-account)
   * Create a Terraform container called `tfstate` inside the new Storage Account
   * Configure Terraform tfstate backend to Azurerm - [Link](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)
 * Add the following details to the `batch/serverless_functions/aws/backend.conf` file
 ```
 resource_group_name  = "INSERT_RESOUCE_GROUP_NAME_HERE"
 storage_account_name = "INSERT_ADLS_STORAGE_ACCOUNT_NAME_HERE"
 container_name       = "tfstate"
 key                  = "prod.terraform.state"
 ```
{{</ details >}}

{{< details header="Optional" markdown="true" >}}
 * Install [Azure Tools for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack) for creating function templates and accelerating build and debug purposes
{{</ details >}}

## Install

Clone this repository to local computer **[py-superhero-dataflow](https://github.com/jg-ghub/py-superhero-dataflow)** {{< svg-icon "github-icon" >}}
```bash
git clone https://github.com/jg-ghub/py-superhero-dataflow
```

## Services
{{< details header="Schematic" icon="icons/flow-icon.svg" >}}
  {{< get-html "content/data-flow/batch/azure/worker-schematic.html" >}}
{{</ details >}}

### Batch Data Ingestion Worker

Similar to the [Original Kafka Worker](/pub-sub/kafka#worker), this worker is consuming messages from Kafka Topics to clean up stale records in Redis, and also persist the log messages to an ADLS Storage Account in Azure called `raw`. This new worker watches a specific Kafka Topic like `lib.server.lobby`, and is triggered each time it receives a new message. However, unlike the Original Worker which passes individual log messages as JSON to the data sink, this Worker will poll the Topic until a desired amount of un-processed messages have been established to pass log messages as JSON in bulk - desired amount configurable with the `BATCH_SIZE` ENV variable.

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

A serverless function is code that can be executed in a cloud environment without the developer needing to manage the underlying server infrastructure. Serverless computing is a cloud computing paradigm where cloud providers automatically manage the allocation of resources and scaling of applications, allowing developers to focus solely on writing and deploying code. Our Serverless Function in this demonstration utilizes [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview?pivots=programming-language-python) with a [ADLS Blob Trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scenarios?pivots=programming-language-python#process-file-uploads) configuration to execute our ETL job with each JSON log file delivered to our *raw* data sink. The ETL job is fairly simple, it's 
 * **Extract** JSON log files from the ADLS Storage Account *raw* data sink
 * **Transform** the log data to a columnar data structure with respect to the log model (`lib.server.lobby`, `'lib.server.game`, `log.meta`, etc)
 * Finally **Load** the data to Parquet in the *standard* ADLS Storage Account data sink.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fsource%2Fdatasim-function%2F__init__.py%23L85-137&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

The unique aspect to this ETL worker within the Azure Platform can be seen within the Extract and load functions. Specifically, we're using [`adlfs`]("https://github.com/fsspec/adlfs") to not only reads/writes the data to ADLS Storage Accounts, but it's also handling [Authentication](https://github.com/fsspec/adlfs#setting-credentials) for us automatically to pass credentials with ADLS private Storage Accounts from within Azure Functions. This helps to secure the process and adhere to security protocols while eliminating the need to store sensitive credentials close to the source.

#### Local Testing

Replicating the Azure Function invocation locally is encouraged as part of the core function's development. A [Local Storage Emulator](https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-local#local-storage-emulator) can be configured in the local settings configuration file to practice run/debug the core functionality in a local environment without the need of first creating a Storage Account. Microsoft makes creating your first Azure function quite easy with the [Azure Tools for Visual Studio Code Extension](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration), which helps provide an integrated UI to quickly generate a skeleton function to build upon. More details can be found [Here](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-vs-code-python?pivots=python-mode-configuration#create-an-azure-functions-project) on creating your function locally.

## Objective

In this demonstration, we want to evaluate exactly how much infrastructure is required to spin up a Batch Data Ingestion Sink, and an ETL pipeline to standardize JSON log data into a Parquet data structures suitable for Data Warehousing and Advance Analytics in the future. The goal is to be able to benchmark both qualitatively and quantitatively how Azure stacks up against other cloud platforms like AWS and GCP. We will leverage the [Pub/Sub & Message Queue](#pubsub--message-queue) mentioned earlier to ingest data into Azure for our demonstration.

### Infrastructure

The contents of the IaaC for this demonstration is structured with Terraform Modules under the style guide from [Hashicorp Standard Module Structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure). Each component of this deployment is a nested module with custom configurations tailored for our solution.

**Note** Unlike AWS/GCP, Azure has no IP Whitelisting nor Service Account Proxy as added layers of protection for the deployment pipeline, so any account compromise could lead to serious malicious activity to your Azure Account! Please use extra caution when working with Azure. 

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script src="https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fmain.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Security Groups

Create a security group to later bind access rights to the *raw* ADLS Storage Account, and assign membership to the user account responsible for the Batch Data Ingestion with this group. Creating security groups brings efficiency and organization to resource access by defining groups with specific business purposes and limited access to relevant resources. This could include grouping users by business function (like engineer, analyst), or by geographical location (North America vs Europe), etc. However, it's important to distinguish draw backs from assigning members to security groups in Terraform - membership needs to be assigned completely in Terraform or in the Portal. If members were to be assigned originally by Terraform, and then later added by portal, then the next deployment won't take into consideration those members added after-the-fact via the portal, and will eventually lose privileges/access.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fmodules%2Fsuperhero_security_group%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Buckets

This module will create an ADLS Gen2 Storage Account with a single Storage Container (Root Directory) inside. It will also bind the Security Group created with the Contributor Role to this Storage Account. Sometimes having Full Data Ownership can still be [Problematic and raise Authorizations Errors](https://stackoverflow.com/questions/52769758/azure-blob-storage-authorization-permission-mismatch-error-for-get-request-wit).

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fmodules%2Fsuperhero_buckets%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

### Function Zip

In order to include dependencies for our core function, we need to zip the contents of package installation with a [very specific folder structure](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python?tabs=asgi%2Capplication-level&pivots=python-mode-configuration#folder-structure). This method will bypass the deployment installation steps, and instead provide the core functionality and dependencies in the same zip folder to Azure Functions. Trying to [install dependencies at deployment](https://stackoverflow.com/questions/62903172/functionapp-not-importing-python-module) is incredibly problematic, and it's advised to go the ZipDeploy route.

This module will install all the dependencies listed in `requirements.txt` in a specific site-packages child folder path inside the zip file. This zip file will include all function source files in their own directory with a shared dependency library across all functions. Lastly, the zipped package is uploaded to the *functions* Storage Account and an SAS Access Endpoint is captured to provide back to the Function so it has authorized access to the zipped package. This approach would likely be a more efficient route compared to [AWS Lambda Layers](/data-flow/batch/aws#function-layers) if Microsoft dedicated this method as the primary deployment method for Azure Function Apps with Python run times. Unfortunately, this is not the case...

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fmodules%2Fsuperhero_functions_zip%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

#### Function

Much of the heavy lifting is actually done in the Function Zip module. Now the Function module creates a Function App Service Plan with a specific OS Type and resource capacity. Now, Azure Function Apps can be assigned to the Service Plan and configured to Application Insights to log and monitor the Functions execution. By providing the Zipped package's SAS Url to the Function App setting under `WEBSITE_RUN_FROM_PACKAGE`, the Function App will deploy all functions as independent Function App Functions at one instead of deploying each Function App Function one at a time.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fserverless_functions%2Fazure%2Fmodules%2Fsuperhero_functions%2Fresources.tf&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

### Production Deployment

Spinning up this Data flow with Batch Data Ingestion to Azure ADLS Storage Accounts and an Azure Function ETL Job to standarize our Data Simulator Log data to Parquet can be accomplished with the following deployment commands

```bash
terraform -chdir=batch/serverless_functions/azure init -backend-config=$(pwd)/batch/serverless_functions/azure/backend.conf &&
terraform -chdir=batch/serverless_functions/azure plan &&
terraform -chdir=batch/serverless_functions/azure apply -auto-approve
```

```bash
export $(cat batch/kafka/env/kafka.env) && \
export AZURE_ACCOUNT_NAME=$(terraform -chdir=batch/serverless_functions/azure output -raw raw_bucket_name) && \
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.azure.yml build &&
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.azure.yml up -d zookeeper kafka kafka-ui

#Wait for Kafka-ui to be up and connected to Kafka cluster
export PLAYERS=10
docker compose --env-file batch/kafka/env/kafka.env \
  -f batch/kafka/compose/docker-compose.azure.yml up --scale player=${PLAYERS}
```



In this demonstration, we've implemented a data architecture within the cloud environment to construct a scalable foundation for Data Ops. Our chosen cloud platform was Azure, where we utilized ADLS Storage Account data repositories for storing data. Additionally, we employed an ETL Job within Azure Functions to format log data into Parquet into a standardized layer. The process of Data Ingestion was run locally by our Data Simulator application. This application conducted batch processing on Kafka messages, resulting in JSON logs that were directed to ADLS Storage Accounts.

To adhere to Data Governance requirements, we established Security Groups within Azure. These groups were utilized to control access to designated Azure Buckets, thus restricting data accessibility to users and resources with specific use cases. In our instance, this meant facilitating the staging of data from JSON to Parquet. Regrettably, we encountered a limitation in enhancing security measures to safeguard against potential compromises of both our newly created user and the deployment pipeline due to lacking security features.

**Qualitatively** my opinion in regard to how Azure performed in this exercise was pretty poor due to unorganized training materials with both a buggy deployment and randomly failing batch ingestion. The [Azure Tools](https://marketplace.visualstudio.com/items?itemName=ms-vscode.vscode-node-azure-pack) extension in Visual Studio Code was surprisingly very simple to use, and made debugging the Function locally super easy. However, the infra deployment and production development went downhill immediately. The documentation on [creating/deploying an Azure Function](https://learn.microsoft.com/en-us/azure/azure-functions/create-first-function-cli-python?tabs=linux%2Cbash%2Cazure-cli&pivots=python-mode-configuration) and [Configuring a Blob Trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob-trigger?tabs=python-v2%2Cin-process&pivots=programming-language-python) assume that the Function will only be deployed using Azure Tools instead of commonly used IaaC tools like Terraform which caused a lot of frustration trying to get this deployment to work. Buried deep in the Azure documentation are steps on deploying the Functions App infrastructure with the [Zip Deployment Methodology](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python?tabs=asgi%2Capplication-level&pivots=python-mode-configuration) which eventually led us to a working example of this demonstration... Work was given up for a while, as this exercise felt very futile.

Another issue which we still face is the problematic uploads of JSON log data to *raw* ADLS Storage Account. When troubleshooting the application, quickly tearing down and spinning up the application leads to application failure with the exception [Authorization Permission Mismatch](https://stackoverflow.com/questions/52769758/azure-blob-storage-authorization-permission-mismatch-error-for-get-request-wit) for no known reason. The only remedy we've found is to wait longer periods of time between spin-up and tear down of the infrastructure, but really eats up time trying to develop against Azure.

**Quantitatively** our Data Ingestion Pipeline and Serverless ETL Function was a success. The Data Ops were satisfied with this build, and running the experiment costed us **little** money during the trials. Deployment time is fairly quick taking ~5 minutes to deploy and ~5 minutes to tear down. Following the [Don't Repeat Yourself (DRY) Guidelines](https://terragrunt.gruntwork.io/docs/features/keep-your-terraform-code-dry/), we were able to build this deployment pipeline with 4 distinct types of modules for a total of ~300+ lines of Terraform code. However, with the reduction in code length comes less security mechanisms around account compromise.

Azure was a difficult platform to work with in this specific scenario, but it eventually met all of our expectations to get this demonstration up and running. We hope if you ever intend to work with Azure, or are currently working with Azure and are trying to build a similar capability, that you accelerate your development by using this template we've built. Save yourself from some frustration...