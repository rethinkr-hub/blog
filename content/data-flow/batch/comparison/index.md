---
title: "Dataflow - Data Persistence and Introduction to Data Lake"
date: "2023-08-09"
draft: false
tags: ["Dataflow", "Data Lake", "Batch Ingestion", "Data Sink"]
author: "Jason Grein"
description: "Comparing Storage Providers with our Batch Data Ingestion Pipelines"

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

In this series, we'll begin to dive deeper into Big Data topics like Cloud Computing & Big Data Platforms, Data Integration and Data Modelling. Cloud Computing & Big Data Platforms enable organizations to scale and process vast amounts of data efficiently, fostering innovation and data-driven decision-making. Data Integration ensures seamless amalgamation of heterogeneous data sources, facilitating comprehensive insights, while Data Modelling structures data for analysis, enhancing understanding, and enabling predictive and prescriptive analytics, collectively empowering businesses to harness the full potential of their data, gain competitive advantages, and achieve strategic goals.

**Motivation**: design a mutli-cloud Big Data Integration system to test the limits of speed-to-deployment on popular cloud providers with an interchangeable ETL pipeline. This approach will allow us to easily switch between cloud techs, ensuring simplicity and flexibility in our system architecture.

## Prerequisites

### Terraform

Cloud infrastructure refers to the virtualized resources and services offered by cloud computing providers, such as [Amazon Web Services (AWS)](https://aws.amazon.com/), [Microsoft Azure](https://azure.microsoft.com/), or [Google Cloud Platform (GCP)](https://cloud.google.com/), that allow businesses and individuals to deploy and manage their applications, data, and services over the internet. It encompasses a variety of components, including computing power (virtual machines or containers), storage, networking capabilities, and databases, all of which can be provisioned and scaled up or down on-demand to meet specific workload requirements. Cloud infrastructure offers benefits like flexibility, scalability, cost efficiency, and global accessibility, enabling organizations to focus on their core functions without the burden of managing physical hardware.

[Terraform](https://www.terraform.io/), on the other hand, is an infrastructure-as-code (IaC) tool that facilitates the provisioning and management of cloud infrastructure through declarative configuration files. By defining infrastructure components and their relationships in Terraform's domain-specific language, users can create, update, and delete resources across different cloud providers consistently and reproducibly. Terraform abstracts the complexity of interacting with various cloud APIs and handles the provisioning process, ensuring that the declared state of the infrastructure matches the actual state, which enhances collaboration, version control, and automated deployment processes. In summary, cloud infrastructure combined with Terraform offers a streamlined, programmable approach to building and managing scalable and dynamic IT environments in the cloud.

This series will require Terraform CLI to be installed on your workstation. Please visit [Terraform Installation Docs](https://developer.hashicorp.com/terraform/downloads) in order to install on your machine.

## Design Purposefully for Multi-Cloud

Multi-cloud within DataOps refers to the strategy of utilizing multiple cloud service providers for managing and processing data operations within an organization. This approach aims to enhance flexibility, scalability, and resilience by distributing data and workloads across different cloud environments, mitigating vendor lock-in, and enabling efficient resource allocation based on specific requirements. It involves orchestrating data pipelines, storage, analytics, and other data-related processes seamlessly across diverse cloud platforms, optimizing data management practices and maximizing the benefits of cloud computing while minimizing potential risks and dependencies associated with a single cloud provider.

### Data Lake Layer Architecture

Data lake layers are organizational structures used in a data lake architecture to manage and structure data efficiently. They help in organizing, securing, and maintaining the data stored in a data lake. These layers represent different stages of data processing and refinement. Layers help in managing the data lifecycle and ensuring that data is processed and made available in a controlled manner. In our Data Lake, we will use the following layer to define our data's life cycle:

 * **Raw Layer** is the landing area for all incoming data, regardless of its format or source. It is often used for initial storage before any processing or transformation occurs. Data in this layer is typically stored in its raw and unaltered form.

 * **Standard Layer** is usefull for transformimg all raw assets into a standardized data format (like JSON to Parquet) for later cleaning and repurposing. This allows us to establish a raw level schema for our data, and create a base version for data lineage.

 * **Stage Layer** is where data is organized and structured for specific use cases. It's where data is transformed into a more usable and queryable format. Data quality, metadata, and governance processes are applied to make the data reliable and consistent.

 * **Output Layer** makes data available for end-users, analysts, and applications. It can include various layers for different types of users or use cases. Access control and security measures are implemented to ensure data is only accessible to authorized users.

 * **Archival Layer** belongs to data that is no longer actively used where the asset may be moved to allowing a reduction in storage costs. This layer typically contains historical data that may be needed for compliance or reference purposes.

In this Dataflow exercise, we focus on the first two layers of the Data Lake Layer Architecture. This will help us to organize a raw data sink for data ingestion pipelines to safely store source data, and also standardize the analytical data format as Parquet to reduce the need for multiple data processing techniques/tools.

### Unified Filesystem
In order to reach this multi-cloud strategy, we leveraged [FSSpec](https://filesystem-spec.readthedocs.io/en/latest/), and respective Cloud Filesystem libraries, to handle the different cloud storage complexities like authentication and internal mappings. This helps to provide a simple to use and familiar API to the like of Python's [File Object](https://docs.python.org/3/glossary.html#term-file-object).

Filesystem libraries are baked into the worker service which allows us to seamlessly switch between cloud provider given the change in ENV Variable `OUTPUT_FS`. Each cloud provider has a very simple Filesystem which we configure to the `filesystem` variable, and acts identical to the [File Object](https://docs.python.org/3/glossary.html#term-file-object). We also keep security top of mind in this pattern by ensuring the `filesystem` creation is utilizing its respective cloud CLI to authenticate, in contrast to storing credentials close to the source code.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fkafka%2Flib%2Fworker%2Fparser.py%23L1-L62&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

### Kafka Batch
Under normal circumstances Kafka is better utilized for streaming data, but in this demonstration we can configure Kafka to operate with batch data ingestion pipelines to cloud sinks. We can borrow from the [AIOKafka Local State Consumer Template](https://aiokafka.readthedocs.io/en/stable/examples/local_state_consumer.html), which creates a local state tracking system to monitor topic partition message counts, and switches the consumer feature of the workers to trigger less frequently with larger offset message intakes. This allows us to process messages in Kafka topics in batches, which in this demonstration makes sense to run ETL tasks instead of running an ETL job per message. 

In addition, we can build on the published work by AIOKafka to store the topic partition message counts in Redis Mem Cache instead of locally within the application which can aid us with scalability in the future. We implement a re-balancer class called RebalanceListener to dump all counts and offsets before Kafka triggered re-balances. After the re-balance sequence is completed, the state is loaded back from redis. Besides monitoring for re-balances, we also have a separate state interface with redis running on its own loop to save the local state tracking system every second.

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fkafka%2Flib%2Futils%2Fkafka%2Frebalance.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

Now we can configure the consumer not to store commit points in Kafka's internal storage, as that will be managed by Redis state. Instead, the consumer can `seek` message offsets to pull a bundle of messages at a time in a batch like fashion, which will happen once the internal count surpasses the `BATCH_SIZE`. All of this requires the consumer preferences bet switched to
 * `enable_auto_commit=False`
 * `auto_offset_reset="none"`

{{< details header="Github Code" icon="icons/github-icon.svg" >}}
  {{< get-script "https://emgithub.com/embed-v2.js?target=https%3A%2F%2Fgithub.com%2Frethinkr-hub%2Fpy-superhero-dataflow%2Fblob%2Fmain%2Fbatch%2Fkafka%2Flib%2Fpubsub%2Fkafka.py&style=github-dark&type=code&showBorder=on&showLineNumbers=on&showFileMeta=on&showFullPath=on&showCopy=on" >}}
{{</ details >}}

## Objective

Use the Data Simulator and new Kafka batch capabilities to evaluate how reliable, efficient and cost effective each popular Cloud Platform is in terms of handling a data flow from source application to data sink. We will utilize the [Unified Filesystem](#unified-filesystem) and [Kafka Batch](#kafka-batch) to ingest log data to blob storage with a common ETL Serverless Function to standardize the raw data to a heterogeneous Parquet file structure.

### Cloud Platforms

#### Amazon Web Services (AWS)

{{< refcard "/data-flow/batch/aws" >}}

In this article, we explore using AWS for hosting the infrastructure to complete the Dataflow objective at hand. AWS provides a well-established platform for creating a data sink with S3, and super quickly triggering a staging ETL job with AWS Lambda Serverless Functions with a S3 Blob Trigger. A great benefit about using AWS with security in mind, we're able to add an extra layer of protection around the IAC deployment pipeline by introducing a custom IAM Policy to only allow our users/service principals access to AWS Account resources via IP Whitelisting. This feature isn't commonly available in other Cloud Platforms. Continue reading the article for more details on other pros & cons we faced when using AWS for this exercise.

#### Azure

{{< refcard-alt "/data-flow/batch/azure" >}}

This demonstration looks to Azure for hosting the instrastucture required to complete our Dataflow experiment. According to Microsoft, Azure boasts the [Largest Market Share Among Fortune 500 Companies at 95%!](https://azure.microsoft.com/en-ca/resources/cloud-computing-dictionary/what-is-azure/), so it's no suprise that Azure has a big adoption rate and a top player in the Cloud Platform space. In this excersize we were able to create a data sink using ADLS Gen2 Storage Accounts, and a staging ETL job with Azure Functions App with a Storage Account Blob Trigger. Unfortunately our trial has left us with a sour opinion on this Platform because development time was excessively long when trying to deploy a working Blob Trigger Function to Azure Functions - too much trial and error due to poor documentation from our experience. Continue reading the article for more details on how Azure stacks up to the Dataflow objective.

#### Google Cloud Platform (GCP)

{{< refcard "/data-flow/batch/gcp" >}}

This last article uses GCP for hosting the infrastructure necessary to complete our Dataflow experiment. GCP was the latest Platform to join the big three, but is very quickly gaining adoption and market share against AWS and Azure... and for good reasons. GCP was an excellent choice for taking care of this Datflow pipeline. We were able to create a data sink using GCS Buckets and build a staging ETL job with Cloud Functions using a GCS Blob Trigger. The added secruity around the Deployment Pipeline using their Service Principal Impersonator feature to generate authentication tokens allowed us to interact with GCP APIs without the need to ever generat a security key. Continue reading the article for more details on how GCP performed against the Dataflow objective.