---
title: Docker containers 
titleSuffix: Language Understanding - Azure Cognitive Services
description: The LUIS container loads your trained or published app into a docker container and provides access to the query predictions from the container's API endpoints. 
services: cognitive-services
author: diberry
manager: nitinme
ms.custom: seodec18
ms.service: cognitive-services
ms.subservice: language-understanding
ms.topic: article
ms.date: 01/22/2019
ms.author: diberry
---

# Install and run LUIS docker containers
 
The Language Understanding (LUIS) container loads your trained or published Language Understanding model, also know as a [LUIS app](https://www.luis.ai), into a docker container and provides access to the query predictions from the container's API endpoints. You can collect query logs from the container and upload these back to the Azure Language Understanding model to improve the app's prediction accuracy.

The following video demonstrates using this container.

[![Container demonstration for Cognitive Services](./media/luis-container-how-to/luis-containers-demo-video-still.png)](https://aka.ms/luis-container-demo)

If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.

## Prerequisites

In order to run the LUIS container, you must have the following: 

|Required|Purpose|
|--|--|
|Docker Engine| You need the Docker Engine installed on a [host computer](#the-host-computer). Docker provides packages that configure the Docker environment on [macOS](https://docs.docker.com/docker-for-mac/), [Windows](https://docs.docker.com/docker-for-windows/), and [Linux](https://docs.docker.com/engine/installation/#supported-platforms). For a primer on Docker and container basics, see the [Docker overview](https://docs.docker.com/engine/docker-overview/).<br><br> Docker must be configured to allow the containers to connect with and send billing data to Azure. <br><br> **On Windows**, Docker must also be configured to support Linux containers.<br><br>|
|Familiarity with Docker | You should have a basic understanding of Docker concepts, like registries, repositories, containers, and container images, as well as knowledge of basic `docker` commands.| 
|Language Understanding (LUIS) resource and associated app |In order to use the container, you must have:<br><br>* A [_Language Understanding_ Azure resource](luis-how-to-azure-subscription.md), along with the associated endpoint key and endpoint URI (used as the billing endpoint).<br>* A trained or published app packaged as a mounted input to the container with its associated App ID.<br>* The Authoring Key to download the app package, if you are doing this from the API.<br><br>These requirements are used to pass command-line arguments to the following variables:<br><br>**{AUTHORING_KEY}**: This key is used to get the packaged app from the LUIS service in the cloud and upload the query logs back to the cloud. The format is `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.<br><br>**{APPLICATION_ID}**: This ID is used to select the App. The format is `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.<br><br>**{ENDPOINT_KEY}**: This key is used to start the container. You can find the endpoint key in two places. The first is the Azure portal within the _Language Understanding_ resource's keys list. The endpoint key is also available in the LUIS portal on the Keys and Endpoint settings page. Do not use the starter key.<br><br>**{BILLING_ENDPOINT}**: The billing endpoint value is available on the Azure portal's Language Understanding Overview page. An example is: `https://westus.api.cognitive.microsoft.com/luis/v2.0`.<br><br>The [authoring key and endpoint key](luis-boundaries.md#key-limits) have different purposes. Do not use them interchangeably. |

### The host computer

[!INCLUDE [Request access to private preview](../../../includes/cognitive-services-containers-host-computer.md)]

### Container requirements and recommendations

This container supports minimum and recommended values for the settings:

|Setting| Minimum | Recommended |
|-----------|---------|-------------|
|Cores<BR>`--cpus`|1 core|1 core|
|Memory<BR>`--memory`|2 GB|4 GB|
|Transactions per second<BR>(TPS)|20 TPS|40 TPS|

Each core must be at least 2.6 gigahertz (GHz) or faster.

The `--cpus` and `--memory` settings are used as part of the `docker run` command.

## Get the container image with `docker pull`

Use the [`docker pull`](https://docs.docker.com/engine/reference/commandline/pull/) command to download a container image from the `mcr.microsoft.com/azure-cognitive-services/luis` repository:

```Docker
docker pull mcr.microsoft.com/azure-cognitive-services/luis:latest
```

Use the [`docker pull`](https://docs.docker.com/engine/reference/commandline/pull/) command to download a container image.

For a full description of available tags, such as `latest` used in the preceding command, see [LUIS](https://go.microsoft.com/fwlink/?linkid=2043204) on Docker Hub.

[!INCLUDE [Tip for using docker list](../../../includes/cognitive-services-containers-docker-list-tip.md)]


## How to use the container

Once the container is on the [host computer](#the-host-computer), use the following process to work with the container.

![Process for using Language Understanding (LUIS) container](./media/luis-container-how-to/luis-flow-with-containers-diagram.jpg)

1. [Export package](#export-packaged-app-from-luis) for container from LUIS portal or LUIS APIs.
1. Move package file into the required **input** directory on the [host computer](#the-host-computer). Do not rename, alter, or decompress LUIS package file.
1. [Run the container](##run-the-container-with-docker-run), with the required _input mount_ and billing settings. More [examples](luis-container-configuration.md#example-docker-run-commands) of the `docker run` command are available. 
1. [Querying the container's prediction endpoint](#query-the-containers-prediction-endpoint). 
1. When you are done with the container, [import the endpoint logs](#import-the-endpoint-logs-for-active-learning) from the output mount in the LUIS portal and [stop](#stop-the-container) the container.
1. Use LUIS portal's [active learning](luis-how-to-review-endoint-utt.md) on the **Review endpoint utterances** page to improve the app.

The app running in the container can't be altered. In order the change the app in the container, you need to change the app in the LUIS service using the [LUIS](https://www.luis.ai) portal or use the LUIS [authoring APIs](https://westus.dev.cognitive.microsoft.com/docs/services/5890b47c39e2bb17b84a55ff/operations/5890b47c39e2bb052c5b9c2f). Then train and/or publish, then download a new package and run the container again.

The LUIS app inside the container can't be exported back to the LUIS service. Only the query logs can be uploaded. 

## Export packaged app from LUIS

The LUIS container requires a trained or published LUIS app to answer prediction queries of user utterances. In order to get the LUIS app, use either the trained or published package API. 

The default location is the `input` subdirectory in relation to where you run the `docker run` command.  

Place the package file in a directory and reference this directory as the input mount when you run the docker container. 

### Package types

The input mount directory can contain the **Production**, **Staging**, and **Trained** versions of the app simultaneously. All the packages are mounted. 

|Package Type|Query Endpoint API|Query availability|Package filename format|
|--|--|--|--|
|Trained|Get, Post|Container only|`{APPLICATION_ID}_v{APPLICATION_VERSION}.gz`|
|Staging|Get, Post|Azure and container|`{APPLICATION_ID}_STAGING.gz`|
|Production|Get, Post|Azure and container|`{APPLICATION_ID}_PRODUCTION.gz`|

> [!IMPORTANT]
> Do not rename, alter, or decompress the LUIS package files.

### Packaging prerequisites

Before packaging a LUIS application, you must have the following:

|Packaging Requirements|Details|
|--|--|
|Azure _Language Understanding_ resource instance|Supported regions include<br><br>West US (```westus```)<br>West Europe (```westeurope```)<br>Australia East (```australiaeast```)|
|Trained or published LUIS app|With no [unsupported dependencies](#unsupported-dependencies). |
|Access to the [host computer](#the-host-computer)'s file system |The host computer must allow an [input mount](luis-container-configuration.md#mount-settings).|
  
### Export app package from LUIS portal

The LUIS [portal](https://www.luis.ai) provides the ability to export the trained or published app's package. 

### Export published app's package from LUIS portal

The published app's package is available from the **My Apps** list page. 

1. Sign on to the LUIS [portal](https://www.luis.ai).
1. Select the checkbox to the left of the app name in the list. 
1. Select the **Export** item from the contextual toolbar above the list.
1. Select **Export for container (GZIP)**.
1. Select the environment of **Production slot** or **Staging slot**.
1. The package is downloaded from the browser.

![Export the published package for the container from the App page's Export menu](./media/luis-container-how-to/export-published-package-for-container.png)

### Export trained app's package from LUIS portal

The trained app's package is available from the **Versions** list page. 

1. Sign on to the LUIS [portal](https://www.luis.ai).
1. Select the app in the list. 
1. Select **Manage** in the app's navigation bar.
1. Select **Versions** in the left navigation bar.
1. Select the checkbox to the left of the version name in the list.
1. Select the **Export** item from the contextual toolbar above the list.
1. Select **Export for container (GZIP)**.
1. The package is downloaded from the browser.

![Export the trained package for the container from the Versions page's Export menu](./media/luis-container-how-to/export-trained-package-for-container.png)


### Export published app's package from API

Use the following REST API method, to package a LUIS app that you've already [published](luis-how-to-publish-app.md). Substituting your own appropriate values for the placeholders in the API call, using the table below the HTTP specification.

```http
GET /luis/api/v2.0/package/{APPLICATION_ID}/slot/{APPLICATION_ENVIRONMENT}/gzip HTTP/1.1
Host: {AZURE_REGION}.api.cognitive.microsoft.com
Ocp-Apim-Subscription-Key: {AUTHORING_KEY}
```

| Placeholder | Value |
|-------------|-------|
|{APPLICATION_ID} | The application ID of the published LUIS app. |
|{APPLICATION_ENVIRONMENT} | The environment of the published LUIS app. Use one of the following values:<br/>```PRODUCTION```<br/>```STAGING``` |
|{AUTHORING_KEY} | The authoring key of the LUIS account for the published LUIS app.<br/>You can get your authoring key from the **User Settings** page on the LUIS portal. |
|{AZURE_REGION} | The appropriate Azure region:<br/><br/>```westus``` - West US<br/>```westeurope``` - West Europe<br/>```australiaeast``` - Australia East |

Use the following CURL command to download the published package, substituting your own values:

```bash
curl -X GET \
https://{AZURE_REGION}.api.cognitive.microsoft.com/luis/api/v2.0/package/{APPLICATION_ID}/slot/{APPLICATION_ENVIRONMENT}/gzip  \
 -H "Ocp-Apim-Subscription-Key: {AUTHORING_KEY}" \
 -o {APPLICATION_ID}_{APPLICATION_ENVIRONMENT}.gz
```

If successful, the response is a LUIS package file. Save the file in the storage location specified for the input mount of the container. 

### Export trained app's package from API

Use the following REST API method, to package a LUIS application that you've already [trained](luis-how-to-train.md). Substituting your own appropriate values for the placeholders in the API call, using the table below the HTTP specification.

```http
GET /luis/api/v2.0/package/{APPLICATION_ID}/versions/{APPLICATION_VERSION}/gzip HTTP/1.1
Host: {AZURE_REGION}.api.cognitive.microsoft.com
Ocp-Apim-Subscription-Key: {AUTHORING_KEY}
```

| Placeholder | Value |
|-------------|-------|
|{APPLICATION_ID} | The application ID of the trained LUIS application. |
|{APPLICATION_VERSION} | The application version of the trained LUIS application. |
|{AUTHORING_KEY} | The authoring key of the LUIS account for the published LUIS app.<br/>You can get your authoring key from the **User Settings** page on the LUIS portal.  |
|{AZURE_REGION} | The appropriate Azure region:<br/><br/>```westus``` - West US<br/>```westeurope``` - West Europe<br/>```australiaeast``` - Australia East |

Use the following CURL command to download the trained package:

```bash
curl -X GET \
https://{AZURE_REGION}.api.cognitive.microsoft.com/luis/api/v2.0/package/{APPLICATION_ID}/versions/{APPLICATION_VERSION}/gzip  \
 -H "Ocp-Apim-Subscription-Key: {AUTHORING_KEY}" \
 -o {APPLICATION_ID}_v{APPLICATION_VERSION}.gz
```

If successful, the response is a LUIS package file. Save the file in the storage location specified for the input mount of the container. 

## Run the container with `docker run`

Use the [docker run](https://docs.docker.com/engine/reference/commandline/run/) command to run the container. The command uses the following parameters:

| Placeholder | Value |
|-------------|-------|
|{ENDPOINT_KEY} | This key is used to start the container. Do not use the starter key. |
|{BILLING_ENDPOINT} | The billing endpoint value is available on the Azure portal's Language Understanding Overview page.|

Replace these parameters with your own values in the following example `docker run` command.

```bash
docker run --rm -it -p 5000:5000 --memory 4g --cpus 2 \
--mount type=bind,src=c:\input,target=/input \
--mount type=bind,src=c:\output,target=/output \
mcr.microsoft.com/azure-cognitive-services/luis \
Eula=accept \
Billing={BILLING_ENDPOINT} \
ApiKey={ENDPOINT_KEY}
```

> [!Note] 
> The preceding command uses the directory off the `c:` drive to avoid any permission conflicts on Windows. If you need to use a specific directory as the input directory, you may need to grant the docker service permission. 
> The preceding docker command uses the back slash, `\`, as a line continuation character. Replace or remove this based on your [host computer](#the-host-computer) operating system's requirements. Do not change the order of the arguments unless you are very familiar with docker containers.


This command:

* Runs a container from the LUIS container image
* Loads LUIS app from input mount at c:\input, located on container host
* Allocates two CPU cores and 4 gigabytes (GB) of memory
* Exposes TCP port 5000 and allocates a pseudo-TTY for the container
* Saves container and LUIS logs to output mount at c:\output, located on container host
* Automatically removes the container after it exits. The container image is still available on the host computer. 

More [examples](luis-container-configuration.md#example-docker-run-commands) of the `docker run` command are available. 

> [!IMPORTANT]
> The `Eula`, `Billing`, and `ApiKey` options must be specified to run the container; otherwise, the container won't start.  For more information, see [Billing](#billing).
> The ApiKey value is the **Key** from the Keys and Endpoints page in the LUIS portal and is also available on the Azure Language Understanding Resource keys page.  

## Query the container's prediction endpoint

The container provides REST-based query prediction endpoint APIs. Endpoints for published (staging or production) apps have a _different_ route than endpoints for trained apps. 

Use the host, `https://localhost:5000`, for container APIs. 

|Package type|Method|Route|Query parameters|
|--|--|--|--|
|Published|[Get](https://westus.dev.cognitive.microsoft.com/docs/services/5819c76f40a6350ce09de1ac/operations/5819c77140a63516d81aee78), [Post](https://westus.dev.cognitive.microsoft.com/docs/services/5819c76f40a6350ce09de1ac/operations/5819c77140a63516d81aee79)|/luis/v2.0/apps/{appId}?|q={q}<br>&staging<br>[&timezoneOffset]<br>[&verbose]<br>[&log]<br>|
|Trained|Get, Post|/luis/v2.0/apps/{appId}/versions/{versionId}?|q={q}<br>[&timezoneOffset]<br>[&verbose]<br>[&log]|

The query parameters configure how and what is returned in the query response:

|Query parameter|Type|Purpose|
|--|--|--|
|`q`|string|The user's utterance.|
|`timezoneOffset`|number|The timezoneOffset allows you to [change the timezone](luis-concept-data-alteration.md#change-time-zone-of-prebuilt-datetimev2-entity) used by the prebuilt entity datetimeV2.|
|`verbose`|boolean|Returns all intents and their scores when set to true. Default is false, which returns only the top intent.|
|`staging`|boolean|Returns query from staging environment results if set to true. |
|`log`|boolean|Logs queries, which can be used later for [active learning](luis-how-to-review-endoint-utt.md). Default is true.|

### Query published app

An example CURL command for querying the container for a published app is:

```bash
curl -X GET \
"http://localhost:5000/luis/v2.0/apps/{APPLICATION_ID}?q=turn%20on%20the%20lights&staging=false&timezoneOffset=0&verbose=false&log=true" \
-H "accept: application/json"
```
To make queries to the **Staging** environment, change the **staging** query string parameter value to true: 

`staging=true`

### Query trained app

An example CURL command for querying the container for a trained app is: 

```bash
curl -X GET \
"http://localhost:5000/luis/v2.0/apps/{APPLICATION_ID}/versions/{APPLICATION_VERSION}?q=turn%20on%20the%20lights&timezoneOffset=0&verbose=false&log=true" \
-H "accept: application/json"
```
The version name has a maximum of 10 characters and contains only characters allowed in a URL. 

## Import the endpoint logs for active learning

If an output mount is specified for the LUIS container, app query log files are saved in the output directory, where {INSTANCE_ID} is the container ID. The app query log contains the query, response, and timestamps for each prediction query submitted to the LUIS container. 

The following location shows the nested directory structure for the container's log files.
`
/output/luis/{INSTANCE_ID}/
`
 
From the LUIS portal, select your app, then select **Import endpoint logs** to upload these logs. 

![Import container's log files for active learning](./media/luis-container-how-to/upload-endpoint-log-files.png)

After the log is uploaded, [review the endpoint](https://docs.microsoft.com/azure/cognitive-services/luis/luis-concept-review-endpoint-utterances) utterances in the LUIS portal.

## Stop the container

To shut down the container, in the command-line environment where the container is running, press **Ctrl+C**.

## Troubleshooting

If you run the container with an output [mount](luis-container-configuration.md#mount-settings) and logging enabled, the container generates log files that are helpful to troubleshoot issues that happen while starting or running the container. 

## Container's API documentation

[!INCLUDE [Container's API documentation](../../../includes/cognitive-services-containers-api-documentation.md)]

## Billing

The LUIS container sends billing information to Azure, using a _Language Understanding_ resource on your Azure account. 

Cognitive Services containers are not licensed to run without being connected to Azure for metering. Customers need to enable the containers to communicate billing information with the metering service at all times. Cognitive Services containers do not send customer data (the utterance) to Microsoft. 

The `docker run` uses the following arguments for billing purposes:

| Option | Description |
|--------|-------------|
| `ApiKey` | The API key of the _Language Understanding_ resource used to track billing information.<br/>The value of this option must be set to an API key for the provisioned LUIS Azure resource specified in `Billing`. |
| `Billing` | The endpoint of the _Language Understanding_ resource used to track billing information.<br/>The value of this option must be set to the endpoint URI of a provisioned LUIS Azure resource.|
| `Eula` | Indicates that you've accepted the license for the container.<br/>The value of this option must be set to `accept`. |

> [!IMPORTANT]
> All three options must be specified with valid values, or the container won't start.

For more information about these options, see [Configure containers](luis-container-configuration.md).

## Unsupported dependencies

You can use a LUIS application if it **doesn't include** any of the following dependencies:

Unsupported app configurations|Details|
|--|--|
|Unsupported container cultures| German (de-DE)<br>Dutch (nl-NL)<br>Japanese (ja-JP)<br>|
|Unsupported domains|Prebuilt domains, including prebuilt domain intents and entities|
|Unsupported entities for all cultures|[KeyPhrase](https://docs.microsoft.com/azure/cognitive-services/luis/luis-reference-prebuilt-keyphrase) prebuilt entity for all cultures|
|Unsupported entities for English (en-US) culture|[GeographyV2](https://docs.microsoft.com/azure/cognitive-services/luis/luis-reference-prebuilt-geographyv2) prebuilt entities|
|Speech priming|External dependencies are not supported in the container.|
|Sentiment analysis|External dependencies are not supported in the container.|
|Bing spell check|External dependencies are not supported in the container.|

## Summary

In this article, you learned concepts and workflow for downloading, installing, and running Language Understanding (LUIS) containers. In summary:

* Language Understanding (LUIS) provides one Linux container for Docker providing endpoint query predictions of utterances.
* Container images are downloaded from the Microsoft Container Registry (MCR).
* Container images run in Docker.
* You can use REST API to query the container endpoints by specifying the host URI of the container.
* You must specify billing information when instantiating a container.

> [!IMPORTANT]
> Cognitive Services containers are not licensed to run without being connected to Azure for metering. Customers need to enable the containers to communicate billing information with the metering service at all times. Cognitive Services containers do not send customer data (for example, the image or text that is being analyzed) to Microsoft.

## Next steps

* Review [Configure containers](luis-container-configuration.md) for configuration settings
* Refer to [Frequently asked questions (FAQ)](luis-resources-faq.md) to resolve issues related to LUIS functionality.
* Use more [Cognitive Services Containers](../cognitive-services-container-support.md)
