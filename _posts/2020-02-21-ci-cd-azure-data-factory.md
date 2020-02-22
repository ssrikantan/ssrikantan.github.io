---
layout: post
title: "CI/CD automation of Azure Data factory Pipelines"
date: 2020-02-21
---
## Summary
In this post, I have taken the example of an Azure Data factory pipeline to demonstrate how CI/CD Automation can be implemented. While doing so, I have referred to the documentation here for guidance.
Azure Data factory provides the ability to author data processing pipelines, manage the resulting pipeline as a JSON document that can be source and version controlled. New and updated pipelines from development can be pushed to the staging and Production branches in Azure Devops, from where these could be deployed to the Data factory instances running in the other environments like staging and production.
Scenario Description

Azure SQL Database hosts Customer (Audience) data. A Stored procedure on this database returns the audience that should be targeted with a specific Marketing campaign, over email. The size of the audience data returned by the Stored Procedure is huge, and is stored in Azure blob Storage, in chunks of 4MB JSON Files. Once the data is transferred to the blob Storage account, a single Email is sent to the Administrator notifying the completion of the pipeline run. To send this email, a Web activity in the pipeline is used to invoke an Azure Logic App.
Additionally, an Azure Logic App is triggered by Blob insert events on the storage account, that de-batches the data per user and individually sends the campaign email. The processing of the campaign emails through Logic Apps is not in the scope of this article.
ADF Pipeline activities, Connections and Linked services

Figure 1 shows the ADF Pipeline ‘notifierjob’. The Pipeline is integrated with Azure Devops Git Repository, and linked to the ‘Developer branch’. All changes or new features in the pipeline would be implemented in this branch before it is pushed to staging and production environments.
Pipeline Activities:

1) Copy Activity: Azure SQL Database (Source) (action : execute Stored procedure). Data is copied to Azure Storage Account (Sink)

2) Web activity: Makes a HTTP POST call to an Azure Logic App. The body of the HTTP Request is a JSON, that contains the Recipient email address, Body and subject of the email to the administrator.

![GitHub Logo](../../../images/ADFPipeline.png)

Figure 1 ADF Pipeline

## Linked Services and Connections:

a) Azure Key Vault is used to store the Connection string to access Azure SQL Database (source) and the Azure Storage Account (sink) as secrets. The ADF Pipeline is configured to use these Key Vault secrets connects to the source and sink data sources. When the ARM Template is generated on publishing the ADF Pipeline, Azure Key Vault gets added as a Linked service in it.

b) Azure SQL Database

c) Azure Storage Account

Figure 2 shows how the Managed Identity of the ADF instance is used to connect to Azure Key Vault to retrieve the connection string of the Azure SQL Database. Before use in the ADF pipeline designer, the Managed Identity of the ADF instance needs to be provided access to the Key Vault to perform ‘Get’ and ‘List’ operations on secrets.

**Note:**
While the ADF instance can use Managed Identity to access Azure SQL Database and Azure Storage Accounts as well, I have instead used connection strings that contain SQL Authentication credentials and Storage access keys, that are stored as secrets in the Key Vault.


#![GitHub Logo](../../../images/managedidentity.png)
<img src="../../../images/managedidentity.png" alt="Pipeline" height="350px"/>

Figure 2 using Azure Key Vault to store secrets to connect to Azure SQL Database

## Pushing changes in the ADF Pipeline to Azure Devops

Once the changes are made to the pipeline in the designer view, the developer choses the ‘publish’ option that pushes the changes to the ADF instance running in development. This also triggers the creation of the ARM Template in the Repository branch, comprising:
a) ARMTemplateParametersForFactory.json (which is the parameters JSON) and

b) ARMTemplateParametersForFactory.json (which is the resources JSON)

Once the updated ADF Pipeline is tested and ready in the ‘Development Branch’, the developer creates a ‘pull request’ asking to merge the changes with the ‘Master Branch’. Azure Devops provides the workflow to ensure the changes are reviewed (the new ARM Templates with the changes, in this case) before they are accepted and merged with the ‘Master branch’.
When the merge with the ‘Master Branch’ is complete, automatically, a new branch ‘adf_publish’ is created by Azure Devops if it does not exist, containing the new ARM Template.

### Customising the Parameters Template

The content of the parameters JSON of the ARM Template looks as shown below:
![GitHub logo](../../../images/Parameters.png)

Figure 3: Parameters JSON

By default, unlike as shown in Figure 3 above, the parameter definition for the Logic App URL is not created;  instead its value is stored as a string literal in the Resources JSON. To address this requirement, certain changes are required to be done in the Template that generates the Parameters JSON file. The syntax and steps to customise the template are explained here. A base Template available here is downloaded and custom JSON segments as shown in Figure 4, should be added to it.

![GitHub Logo](../../../images/Template.png)

Figure 4: Customise Template that generates ARM Template

The custom Template file (arm-template-parameters-definition.json) should be stored in the root folder of the ‘Development branch’. After the development of the ADF pipeline is completed and when it is published, the engine that generates the ARM Template ensures that a parameter is created for any activity that contains a property with name 'url', and suffixes the name property of this parameter with ‘-logicappurl.

At the end of this step, an ARM Template is generated where the 'url' property of the Logic App Linked service is parameterised.

## Continuous delivery to Staging environment

In the scenario here, a separate Azure Key Vault instance and Azure Data Factory instance are created for use in the Staging environment. Rest of the services like Azure SQL Database, Azure Storage and Azure Logic App from the development environment have been reused in Staging.
Release Pipeline

The steps to be performed are explained here. For the artefacts to the CD Pipeline, select the Resources JSON and the Parameters JSON from the ARM Template generated in the previous section.
The Release pipeline looks as shown below:
i) Retrieve the secrets from azure Key Vault

ii) Deploy the ARM Template to the ADF instance running in the staging environment using the parameters specific to this environment.

Variables, as shown in Figure 5, are defined to pass the Key vault URL, the Logic app URL, etc for the staging environment.

![GitHub Logo](../../../images/ReleasePipeline.png)

Figure 5: CD Pipeline definition with variables

Prior to running the CD Pipeline, the resources for the staging environment, like ADF Instance, Azure Key Vault, etc were already created. The Managed Identity of the ADF Instance was provided access to the Key vault.
Alternatively, as the first step in the CD Pipeline, a PowerShell or Bash script can be executed that first checks the existence of these resources and their requisite permissions, creates/defines them if they do not.

