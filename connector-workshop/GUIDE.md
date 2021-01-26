![Confluent](images/confluent.png)

# Seemlessly Connect Sources and Sink to Confluent Cloud with Kafka Connect
## <div align="center">Lab Guide</div>

## **Agenda**
1. Login to Confluent Cloud
2. Create an Environment and Kafka Cluster
3. Create a Topic and Cloud Dashboard Walkthrough
4. Create an API Key Pair
5. Enable Schema Registry
6. [Set up: Connect Self-Managed Services to Confluent Cloud](#step-6)
7. Deployment: Connect Self Manged Services to Confluent Cloud
8. Install: Self Managed Debezium PostgreSQL CDC Source Connector
9. Launch: PostgreSQL Connector in Confluent Control Center
10. Fully Managed AWS S3 Sink/Azure Blob Storage Sink/Google CLoud Storage Sink Connectors
11. Confluent Cloud Schema Registry
12. Clean Up Resources
13. Confluent Resources and Futher Testing
 

***

## **Architecture**
![architecture](images/architecture.png)

***

## **Prerequisites**

1. Confluent Cloud Account.
    - Sign-up at https://www.confluent.io/confluent-cloud/tryfree/
    - Once you have signed up and logged in, click on the menu icon at the upper right hand corner, click on “Billing & payment”, then enter payment details under “Payment details & contacts”. A screenshot of the billing UI is included below.

> **Note:** We will create resources during this workshop that will incur costs. When you sign up for a Confluent Cloud account, you will get up to $200 per month deducted from your Confluent Cloud statement for the first three months. This will cover the cost of resources created during the workshop. The lab guide includes a section on how to clean up all resources created.

![billing](images/billing.png)

2. Ports 443 and 9092 need to be open to the public internet for outbound traffic. To check, try accessing the following from your web browser:

    portquiz.net:443 \
    portquiz.net:9092

3. This workshop requires access to a terminal or command line interface. 

4. Git CLI utility. Test if this is installed with the following command:
```bash
git --version
```

5. Docker CLI (required) and Docker Desktop (advised). Download both [here](https://docs.docker.com/get-docker/). 

> **Note**: We will be deploying Confluent Platform services and connecting them to Confluent Cloud. There are multiple ways to install Confluent Platform, which you can view in [On-Premises Deployments](https://docs.confluent.io/platform/current/installation/installing_cp/overview.html). In order to make the set up easier for those running different operating systems during the workshop, we will walk through setting up Confluent Platform using Docker. You can accomplish the steps in this lab guide using any of the other deployment methods.

7. An AWS/Azure/GCP account. We will be creating and using a Confluent Fully Managed Sink Connector and connecting it to object storage. You will need the following:
    - Access keys/credentials.
        - AWS: [Access Keys](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys)
        - Azure: [Manage account access keys](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage?tabs=azure-portal)
        GCP: [Creating and Managing Service Accounts](https://cloud.google.com/iam/docs/creating-managing-service-accounts)
    - Bucket/Container Name - Create the object storage before the workshop and have the name of the bucket/container ready.
    - Region - Note which region you are deploying your object storage resource in. You will need to know during the workshop.
    - IAM Policy configured for bucket access
        - AWS: Follow the directions outlined in [IAM Policy for S3](https://docs.confluent.io/cloud/current/connectors/cc-s3-sink.html#cc-s3-bucket-policy)
        - GCP: Your GCP service account role must have permission to create new objects in the GCS bucket. For example, the Storage Admin role can be selected for this purpose. If you are concerned about security and do not want to use the Storage Admin role, only use the storage.objects.get and storage.objects.create roles. 
            > **Note**: The Storage Object Admin role does not work for this purpose.

***

## **Objective**

Welcome to “Seamlessly Connect Sources and Sinks to Confluent Cloud with Kafka Connect”! In this workshop, we will learn how to connect our external systems to Confluent Cloud using Connectors. Confluent offers 180+ pre-built connectors for you to start using today with no coding or developing required. To view the complete list of 180+ connectors that Confluent offers, please see [Confluent Hub](https://www.confluent.io/hub/).

If you attended the first workshop in our Microservices Series, “Getting Started with Microservices in Confluent Cloud”, we walked through how to apply your microservices use case to the world of event streaming with Confluent Cloud. If you did not attend this previous session, do not worry! While our sessions relate, they are completely independent of each other.

Now, what if you have other systems you want to pull data from or push data to? This can be anything from a database, data warehouse, object storage, or software application. You can easily connect these systems to Kafka using one of the connectors.
 
During the workshop, we will first set up our Confluent Cloud account, including creating our first cluster and topic, and setting up Schema Registry. 

Next, we will set up and deploy 2 different types of connectors: Self Managed and Fully Managed.

- Self Managed Connectors are installed on a self managed Kafka Connect cluster that is connected to your cluster in Confluent Cloud. We will be walking through how to set up a local Connect cluster by downloading Confluent Platform, installing the connector offered by Confluent, and then connecting it to our Kafka cluster running in Confluent Cloud.
- Fully Managed Connectors are available as fully managed and fully hosted in Confluent Cloud. We will be walking through how to launch a fully managed connector in the UI. Note that it can also be launched using the ccloud CLI. 

We will also learn more about Schema Registry and how we can use it in Confluent Cloud to ensure data compatibility and to manage our schemas. 

At the conclusion of this workshop, you will have learned how to leverage both self managed and fully managed connectors to complete your data pipeline and how you can use them with your microservices!

***

## **<a name="step-1"></a>Step 1 - Login to Confluent Cloud**
1. Login to Confluent Cloud at https://confluent.cloud and enter in your Email and Password.

![login](images/login.png)

2. If you are logging in for the first time, you will see a self-guided wizard that walks you through spinning up a cluster. Please minimize this as we will walk through those steps in this workshop. 

***

## **<a name="step-2"></a>Step 2 - Create an Environment and Kafka Cluster**

An environment contains Kafka clusters and its deployed components such as Connect, ksqlDB, and Schema Registry. You have the ability to create different environments based on your company's requirements. We’ve seen companies use environments to separate Development/Testing, Pre-Production, and Production clusters. 

1. Click **+ Add environment**. Specify an **Environment name** and click **Create**.

> **Note**: There is a Default environment ready in your account upon account creation. You can use this Default environment for the purpose of this workshop if you do not wish to create an additional environment.

![new-env](images/new-env.png)

2. Now that we have an environment, click **Create Cluster**.

Confluent Cloud clusters are available in 3 types: **Basic**, **Standard**, and **Dedicated**. Basic is intended for development use cases so we will use that for our workshop today. Basic clusters only support single zone availability. Standard and Dedicated clusters are intended for production use and support Multi-zone deployments. If you are interested in learning more about the different types of clusters and their associated features and limits, refer to the documentation [here](https://docs.confluent.io/current/cloud/clusters/cluster-types.html).

3. Choose the **Basic** Cluster Type. 

> **Note**: Confluent’s single tenant clusters, Dedicated clusters, are only available to customers on a Confluent Cloud commit or on PAYG through your cloud provider’s marketplace.

![create-cluster](images/create-cluster.png)

4. Click **Begin Configuration**.

5. Choose your preferred Cloud Provider, Region, and Availability Zone.
    - Here you have the ability to choose which cloud provider you want (AWS, GCP, or Azure). 
    - **Choose the cloud provider you have your object storage set up with** 
    - **Choose the same region where your object storage resource is deployed**

6. Specify a **Cluster Name**. Any name will work here. 

![create-cluster-name](images/create-cluster-name.png)

7. View the associated Configuration & Cost, Usage Limits, and Uptime SLA information before launching.

8. Click **Launch Cluster**.

![launch-cluster](images/launch-cluster.png)

***

## **<a name="step-3"></a>Step 3 - Create a Topic and Walk Through the Cloud Dashboard**

1. On the left hand side navigation menu, you will see **Cluster overview**. 

This section shows Cluster Metrics, such as Throughput and Storage. This page also shows the number of Topics, Partitions, Connectors, and ksqlDB Applications.  Below is an example of the metrics dashboard once you have data flowing through Kafka. 

![cloud-overview](images/cluster-overview.png)

2. Click on **Cluster Settings**. This is an important tab that should be noted. This is where you can find your cluster ID, bootstrap server, cloud details, cluster type, and capacity limits. 

3. **Important**: Copy and save the bootstrap server name. We will use it later in the workshop.

4. On that same navigation menu, select **Topics** and click **Create Topic**. Refresh the page if your cluster is still spinning up.

5. Enter `dbserver1.inventory.customers` as the **Topic name** and **1** as the Number of partitions, and then click on **Create with defaults**.

`dbserver1.inventory.customers` is the name of the table within the Postgres database we will be setting up in a later section.

Topics have many configurable parameters that dictate how Kafka handles messages. A complete list of those configurations for Confluent Cloud can be found [here](https://docs.confluent.io/cloud/current/using/broker-config.html).  If you are interested in viewing the default configurations, you can view them in the Topic Summary on the right side. 

![new-topic](images/new-topic.png)

6. After creation, the **Topics UI** allows you to monitor production and consumption throughput metrics and the configuration parameters for your topics. When we begin sending messages through, you will be able to view messages and message schemas. 

Creating a Schema will ensure messages are following a desired format and can be critical for Kafka production deployments. To learn more about Schema Registry, please see this document [here](https://docs.confluent.io/cloud/current/client-apps/schemas-manage.html#). We will also be going over Schema Registry during this workshop.

7. Below is a look at our topic, `dbserver1.inventory.customers`, but we need to send data to this topic before we see any metrics. 

![topic-overview](images/topic-overview.png)

***

## **<a name="step-4"></a>Step 4 - Create an API Key Pair**

1. Select **API Access** on the navigation menu. 

2. If this is your first API key within your cluster, click **Create key**. If you have set up API keys in your cluster in the past and already have an existing API key, click **+ Add key**.

![create-cc-api-key](images/create-cc-api-key.png)

3. Select **Global Access**, then click Next.

4. Save your API key and secret - you will need these during the workshop.

5. After creating and saving the API key, you will see this API key in the Confluent Cloud UI in the **API Access** tab. If you don’t see the API key populate right away, refresh the browser. 

![cc-api-keys](images/cc-api-keys.png)

***

## **<a name="step-5"></a>Step 5 - Enable Schema Registry**

A Kafka topic contains messages, and each message is a key-value pair. Either the message key or the message value (or both) can be serialized as JSON, Avro, or Protobuf. A schema defines the structure of the data format. 

Confluent Cloud Schema Registry is used to manage schemas and it defines a scope in which schemas can evolve. It stores a versioned history of all schemas, provides multiple compatibility settings, and allows schemas to evolve according to these compatibility settings. 

We will be exploring Confluent Cloud Schema Registry in more detail towards the end of the workshop. First, we will need to enable Schema Registry within our environment.

1. Return to your environment either by clicking on the Confluent icon at the top left corner and then clicking your environment OR clicking on your environment name at the top left corner where it says *`HOME` > `your environment name` > `your cluster name`*.

![sr-cluster](images/sr-cluster.png)

2. Click on **Schemas**. Select your cloud provider and region, and then click on **Enable Schema Registry**.

![sr-tab](images/sr-tab.png)

3. Next, we will create an API Key for Schema Registry. From here, click on **Settings** and expand **Schema Registry API access**.

4. Click on **+ Create Key** and save your API key and secret - you will also need these during the workshop.

5. **Important**: Make note of where it says *“Make requests to the Schema Registry endpoint using your API key and secret for authentication”*. We will use this endpoint in one of the steps later in the workshop.

## **<a name="step-6"></a>Step 6 -  Set up and Connect Self Managed Services to Confluent Cloud**


Let’s say you have a database, or object storage such as AWS S3, Azure Blob Storage, or Google Cloud Storage, or a data warehouse such as Snowflake - how do you connect these data systems to your microservices architecture?

There are 2 options: 
1) Develop your own connectors using the Kafka Connect framework - this requires a lot of development effort and time.  
2) You can leverage the 180+ connectors Confluent offers out-of-the-box which allows you to configure your sources and sinks to Kafka in a few, simple steps. To view the complete list of connectors that Confluent offers, please see [Confluent Hub](https://www.confluent.io/hub/).

With Confluent’s connectors, your data systems can communicate with your microservices, completing your data pipeline. 

If you want to run a connector not yet available as fully managed in Confluent Cloud, you may run it yourself in a self-managed Kafka Connect cluster and connect it to Confluent Cloud. Please note that Confluent will still support any self managed components. 

Now that we have completed setting up our Confluent Cloud account, cluster, topic, and Schema Registry, this next step will guide you how to configure a local Connect cluster backed by your Kafka cluster in Confluent Cloud that we created in Step 2. 

1. Click on **Connectors**, and then click on **Self Managed**. If you already have existing connectors running, click on **+ Add Connector** first.

Self Managed connectors are installed on a local Connect cluster backed by a source Kafka cluster in Confluent Cloud. This Connect cluster will be hosted and managed by you, and Confluent will fully support it. 

## <div align="center">...</div>
*<div align="center">Image</div>*
## <div align="center">...</div>

2. We will now set up a local Connect cluster by downloading Confluent Platform, installing a self managed Connector, and then connecting to our Kafka cluster in Confluent Cloud. First, clone this repository which contains a `docker-compose.yml`: [Confluent Connector Workshop Repository](https://github.com/wangkel/cp-all-in-one-confluent-workshop). 

```bash
git clone https://github.com/wangkel/cp-all-in-one-confluent-workshop.git
```
This Docker Compose file launches Confluent Platform components (except for Kafka brokers, because we will be using the cluster we created in Confluent Cloud) in containers on your local host, and configures them to connect to Confluent Cloud.

We will be using Docker during this workshop. Alternatively,  you can set up these Confluent Platform components and connect them to Confluent Cloud by installing Confluent Platform as a local install.

The Docker Compose file will launch a local Connect cluster and Confluent Control Center. Control Center is used for monitoring your Confluent Platform deployment and can also be connected to your Confluent Cloud deployment. 

This repository is based on the [cp-all-in-one-cloud](https://docs.confluent.io/platform/current/tutorials/build-your-own-demos.html#cp-all-in-one-cloud) demo that Confluent built. The only difference is that the workshop repo has a Postgres database that will launch for us to use and connect to during the workshop. If you are interested in testing out other Confluent-built demos, please see [Build Your Own Demos — Confluent Documentation](https://docs.confluent.io/platform/current/tutorials/build-your-own-demos.html#).

3. Navigate to the **cp-all-in-one-cloud** directory. 
```bash
cd cp-all-in-one-confluent-workshop/cp-all-in-one-cloud
```

4. Checkout the `6.0.1-post` branch.
```bash
git checkout 6.0.1-post
```

## <div align="center">...</div>

