---
layout: post
title: "Deploy a Nginx web application on ACI"
author: Sriram H. Iyer
---

## Overview

[Azure Container Instance (ACI)](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview) is an on-demand serverless containerization service provided by Azure. ACI provides a very simple way to run the containers in Azure without orchestration.

Azure also provides [Azure Container Registry (ACR)](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-intro) to store and maintain container images in Azure. Some of the features of ACR includes enabling private endpoints, service endpoints, controlling access using native firewall settings, etc. 

In this blog, we will create an ACR, build image in ACR, and deploy ACI using the image from ACR.

## Pre-requisites

We need to have [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) installed for performing deployment.

Otherwise, deploy an Azure VM with Managed Identitiy assigned to Contributor role scoped to the Resource Group in which ACR and ACI will be provisioned.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisriram96%2Fblog%2Fmain%2F_arm-templates%2Fazure-virtual-machine-ubuntu-with-identity-and-docker-preinstallation.json)

## Create an ACR

Login to Azure using the `az login` command. When not using Managed Identity or Service Principal, then you will be prompted for authentication.

```
az login --identitiy -u "<Resource ID of Managed Idenitity>"
```

Example:

<img width="1200" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/c9f3e59e-1099-467d-9a9a-0cd4b6774cd8">

You could execute `az account set --subscription "<subscription ID>"` command in case there are multiple subscriptions.

We can create an ACR by executing the command below. Please make sure that the name of ACR is globally unique.

```
az acr create -n "<name of ACR>" -g "<resource group>" --sku Basic
```

Example:

<img width="779" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/2108a6fb-7016-4bd9-8a61-c58b9b80063d">

## Build an image using Azure CLI

The `az acr build` command is used to build an image in the same way we use the `docker build` command. Therefore, we need to prepare the Dockerfile and the context to build images.

You could download the Dockerfile `examplefile-acr.dockerfile` and context file `example.conf` for this example using `curl` utility.

```
curl -O https://raw.githubusercontent.com/hisriram96/blog/main/_docker/examplefile-acr.dockerfile
curl -O https://raw.githubusercontent.com/hisriram96/blog/main/_docker/example.conf
```

<img width="786" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/8c8484a3-b1ad-4257-824f-1e94716770fd">

We could execute the following command to build an image. This command builds image in the ACR and in the local system.

```
az acr build -r <name of ACR> -t examplerepo/exampleimage:1.0.0 -f examplefile-acr.dockerfile .
```

> Note the similarity in the `az acr build` and `docker build` commands. The `-r` option is used to specify the registory where the image will be stored. The other options `-t` is used to specify the image name and tag name for the image and `-f` is used for specifying the Dockerfile as we do in `docker build` command.

## Deploy an ACI

Now that we have built our image in ACR, we can deploy ACI for running our container using the following command.

```
az container create -n "test-container" -g "test-group" --acr-identity "<resource ID of Managed Identity>" --assign-identity "<resource ID of Managed Identity>" --os-type Linux --cpu 1 --memory 1 --image <name of ACR>.azurecr.io/examplerepo/exampleimage:1.0.0 --ports 80 443 --protocol TCP --restart-policy OnFailure --sku Standard --ip-address Public
```

## Verify accessing the Nginx

Once our ACI is deployed, we can  use the `curl` utility to verify if the web page is accessible using the public IP address of the ACI. We should be able to see the "Hello World" web page which we have configured in our Nginx container.

```
curl -v http://www.example.com --resolve www.example.com:80:127.0.0.1
curl -kv https://www.example.com --resolve www.example.com:443:127.0.0.1
```

<img width="624" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/f17f6ba6-9b2c-4f51-b12c-1b9c30c4c298">

<img width="586" alt="image" src="https://github.com/hisriram96/blog/assets/56336513/b2bbea3f-ff0b-4ea8-8afa-340ee093eb41">
