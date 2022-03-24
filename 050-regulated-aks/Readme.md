
# What The Hack - AKS Regulated - PCI - DSS

## Introduction

The objective ofsetup this hack is to setup a regulated AKS cluster for applications that have PCI-DSS requirements.

![image](https://user-images.githubusercontent.com/18613231/159903199-b2ffcd97-c364-4421-9318-3fe77d555151.png)

## Learning Objectives

In this hack you will start by deploying an Azure environment that has the necessary controls as required by PCI-DSS.

You will then setup an AKS cluster keeping PCI-DSS requirements in mind. Finally, we will deploy a sample applications and look at ways of protecting card holder data.

## Challenges

- Challenge 1: **[Prepare](Student/01-prepare.md)**
   - Setup the Azure Environment
- Challenge 2: **[Regulated AKS and Private Clusters](Student/02-aks_private.md)**
   - Deploy the application in an AKS cluster with strict network requirements
- Challenge 3: **[Secrets and Configuration Management](Student/03-aks_secrets.md)**
   - Harden secret management with the help of Azure Key Vault
- Challenge 4: **[AKS Security](Student/04-aks_security.md)**
   - Explore AKS security concepts such as Azure Policy for Kubernetes
- Challenge 5: **[Protecting Data](Student/05-aks_protect_data.md)**
   - Further protect the application data

You can find the source code and documentation in the resources files provided to you for the hack.

## Prerequisites

- Access to an Azure subscription (owner privilege is required in some exercises)

## Contributors

- Hemant Rathore 
- Saurabh Vartak 
- Yusuf Rangwala

