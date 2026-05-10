# Change Event Streaming

This hands-on lab demonstrates the new **Change Event Streaming (CES)** feature in SQL Server 2025 (and Azure SQL Database), which provides reliable, partitioned change events to **Azure Event Hubs**, enabling push-based integrations without polling or ETL jobs. You will:

* Create a database and configure it for CES.
* Define a stream group and associate it with various tables.
* Build a .NET consumer application to listen for and process changes.
* Execute T-SQL operations that generate change events.
* Observe how CES sends events to Azure Event Hubs in real-time.

To support these labs, your instructor has provisioned the following Azure resources for each attendee:

* An Azure Event Hub namespace and hub to support Change Event Streaming in SQL Server.
* An Azure Storage account and container for maintaining change event checkpoints.

Separately, your instructor has also provided you with the SAS token and connection string needed to access the Event Hub and Azure Storage resources respectively.

> **Note:** If you wish to run these labs after the training event, these instructor-provisioned resources will not be available. Instead, you will need to provision these resources in your Azure subscription. For details, refer to [Configure change event streaming](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/configure) in the CES documentation.

___

▶ [Lab: Create the Sample CES Database](https://github.com/lennilobel/sql2025-workshop-hol-orlando2025/blob/main/HOL/3.%20Change%20Event%20Streaming/1.%20Create%20the%20Sample%20CES%20Database.md)
