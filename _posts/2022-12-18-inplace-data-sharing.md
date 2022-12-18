# Using Microsoft Purview in-place Data Sharing

This post dicusses the pain points that in-place data sharing with Microsoft Purview addresses, in an architecture that implements a Data Mesh.

Enterprises have embarked on a massive digital transformation journey, and key in that has been the endeavor to perform analysis on diverse data generated from their Systems. These are their Line of Business Applications hosted on-premise or in the Cloud, SaaS Applications hosted on different Clouds, Telemetry from their devices or applications running in their factories, or from Retail stores. Given the volume, velocity and the variety of the data sources, Organizations have moved beyond just Data Warehouses, into implementing Data Lakes and performing Big Data Analytics on them.

When organizations are complex and there are a number of stakeholders that own the lifecycle of the data generated in their systems, it has becomes necessary for them adopt a Domain Driven Data Model. In this model, rather a single Central team managing the data lifecycle, each Division/Stakeholder in the organization owns that data, manages that as a Product, complete with the data engineering pipelines, how the data is stored, refreshed and offered to other entities, both internal and external to the Organization. This has been known for some time as the 'Data Mesh' architectural pattern.

## Approach used to Share of Data across Data Domains

Data consumers need access to the Data Products that are maintained by a Data Domain, to perform analytics. The consumers would end up building Data pipelines to copy and store the data themselves, which entails additional expenditure, both compute and redundant storage costs.

## In-place Data Sharing with Microsoft Purview

Microsoft Purview supports in-place Data Sharing for data residing in an Azure Blob Storage container, or in Azure Data Lake Gen2 (ADLS Gen2). A Purview Data share is created in the Data Producer's Subscription, and shared with a User who has access to the Blob Storage/ADLS Gen2in the consumer's  account. Once the handshake is established, the data is accessible to the consumer in their Storage Account directly, as 'read-only', without having to perform data copy.

<img src="../../../images/solution-approach.jpg" alt="solutionapproach" height="500px"/>

The Storage costs are borne by the Data provider, where the data always remains, while only the transaction costs are borne by the Data consumers. A single Data share can be linked to multiple consumers.

The figure below shows the Storage transaction logs on both the consumer(ADLS Gen2 Account - erpsource98) and producer (ADLS Gen2 Account - erptarget98) Storage Accounts. Observe that the data access transactions are recorded only for the consumer, when accessing the data in the source.

<img src="../../../images/data-transaction.jpg" alt="transactions" height="500px"/>

## Access to data events in the Consumer Storage Account

We have now seen how the data in the producer is always directly accessible to the data consumers. However, the systems hosted by the consumer could also require that they have access to data events as and when they occur in the producer storage account, for e.g. when a new data file is added to a folder. 
To consider this requirement, an Azure Function App was created that has a Blob trigger configured on the container in the consumer's storage Account. Files were added to a folder in the producer's Storage Account. Observe in the figure below that the Function app is triggered every time a file is added in the producer's Storage Account.

<img src="../../../images/blobtrigger.jpg" alt="blobtrigger" height="750px"/>

## Summary

Microsoft Purview in-place data share helps organizations implement a Domain Driven Model, provide different consumer domains access to a Data Product without having to invest in running data copy pipelines, and redundantly store the same data. Data consumers have direct read-only access to the data in the producer domain.