# Load Data Into Cosmos DB with ADF

In this lab, you will populate an Azure Cosmos DB container from an existing set of data using tools built in to Azure. After importing, you will use the Azure portal to view your imported data.

> If you have not already completed setup for the lab content see the instructions for [Account Setup](00-account_setup.md) before starting this lab.  This will create an Azure Cosmos DB database and container that you will use throughout the lab. You will also use an **Azure Data Factory (ADF)** resource to import existing data into your container.

## Create Azure Cosmos DB Database and Container

You will now create a database and container within your Azure Cosmos DB account.

1. Navigate to the [Azure Portal](https://portal.azure.com)

1. On the left side of the portal, select the **Resource groups** link.

    ![Resource groups is highlighted](../media/03-resource_groups.jpg "Select the Resource Groups")

1. In the **Resource groups** blade, locate and select the **cosmoslabs** resource group.

    ![The cosmoslabs resource group is highlighted](../media/03-lab_resource_group.jpg "Select the cosmoslabs resource group")

1. In the **cosmoslabs** blade, select the **Azure Cosmos DB** account you recently created.

    ![The Cosmos DB resource is highlighted](../media/03-cosmos_resource.jpg "Select the cosmoslabs resource")

1. In the **Azure Cosmos DB** blade, locate and select the **Overview** link on the left side of the blade. At the top select the **Add Container** button.

    ![Add container link is highlighted](../media/03-add_collection.jpg "Add a new container")

1. In the **Add Container** popup, perform the following actions:

    1. In the **Database id** field, select the **Create new** option and enter the value **ImportDatabase**.

    2. Do not check the **Provision database throughput** option.

        > Provisioning throughput for a database allows you to share the throughput among all the containers that belong to that database. Within an Azure Cosmos DB database, you can have a set of containers which shares the throughput as well as containers, which have dedicated throughput.

    3. In the **Container Id** field, enter the value **FoodCollection**.

    4. In the **Partition key** field, enter the value ``/foodGroup``.

    5. In the **Throughput** field, enter the value ``11000``. *Note: we will reduce this to 400 RU/s after the data has been imported*

    6. Select the **OK** button.

1. Wait for the creation of the new **database** and **container** to finish before moving on with this lab.

## Import Lab Data Into Container

You will use **Azure Data Factory (ADF)** to import the JSON array stored in the **nutrition.json** file from Azure Blob Storage.

You do not need to do Steps 1-4 in this section and can proceed to Step 4 by opening your Data Factory (named importNutritionData with a random number suffix)if you are completing the lab through Microsoft Hands-on Labs or ran the setup script, you can use the pre-created Data Factory within your resource group.

1. On the left side of the portal, select the **Resource groups** link.

    > To learn more about copying data to Cosmos DB with ADF, please read [ADF's documentation](https://docs.microsoft.com/azure/data-factory/connector-azure-cosmos-db).

    ![Resource groups link is highlighted](../media/03-resource_groups.jpg "Select Resource Groups")

1. In the **Resource groups** blade, locate and select the **cosmoslabs** resource group.

1. If you see a Data Factory resource, you can skip to step 4, otherwise select **Add** to add a new resource

    ![A data factory resource is highlighted](../media/03-adf-isntance.png "Review if you have data factory already")

    ![Select Add in the nav bar](../media/03-add_adf.jpg "Add a new resource")

   - Search for **Data Factory** and select it. 
   - Create a new **Data Factory**. You should name this data factory **importnutritiondata** with a unique number appended and select the relevant Azure subscription. You should ensure your existing **cosmoslabs** resource group is selected as well as a Version **V2**. 
   - Select **East US** as the region. Do not select **Enable GIT** (this may be checked by default). 
   - Select **Create**.

        ![The new data factory dialog is displayed](../media/03-adf_selections.jpg "Add a new Data Factory resource")

1. After creation is completed, click **Launch Studio** to open Azure Data Factory Studio.

1. We will be using ADF for a one-time copy of data from a source JSON file to a database in Cosmos DB's SQL API. Select **Transform data** to begin the data copy process.
   
   ![alt text](image-7.png)

1. Create link servvice 
   ![alt text](image-9.png)
   ![alt text](image-10.png)
   ![alt text](image-11.png)

2. Name the source **NutritionJson** and select **SAS URI** as the Authentication method. Use the following SAS URI for read-only access to this Blob Storage container:

   ```
   https://ai102str193837986.blob.core.windows.net/cosmos-sample-data/NutritionData.json?sp=r&st=2025-12-07T14:46:41Z&se=2026-01-30T23:01:41Z&spr=https&sv=2024-11-04&sr=b&sig=KeZ0EMtmuy3gR4yKvEYtadvB94PpZz8W2AITWtF0OYY%3D
   ```
   
3. Configure the source dataset by selecting **Azure Blob Storage** as the data store type.
   
   ![alt text](image.png)

4. Set up the connection to the blob storage using the SAS URI provided above.
   
   ![alt text](image-1.png)

5. Configure the destination by selecting **Azure Cosmos DB (SQL API)** as the data store type. Before proceeding, turn on **Data flow debug** to enable testing and validation of your data transformations.
   
   ![alt text](image-2.png)
   ![alt text](image-13.png)

6. Select your Azure Cosmos DB account and the **ImportDatabase** database with **FoodCollection** container as the destination.
   
   ![alt text](image-3.png)

7. Map the source JSON fields to the Cosmos DB container schema. Ensure the partition key mapping is correct.
   
   ![alt text](image-4.png)

8. Review the pipeline settings and click **Finish** to start the data copy operation. Monitor the progress until completion.
   ![alt text](image-8.png)
   ![alt text](image-5.png)

## Validate Imported Data

The Azure Cosmos DB Data Explorer allows you to view documents and run queries directly within the Azure Portal. In this exercise, you will use the Data Explorer to view the data stored in our container.

You will validate that the data was successfully imported into your container using the **Items** view in the **Data Explorer**.

1. Return to the **Azure Portal** (<http://portal.azure.com>).

1. On the left side of the portal, select the **Resource groups** link.

    ![Resource groups link is highlighted](../media/03-resource_groups.jpg "Select your resource group")

1. In the **Resource groups** blade, locate and select the **cosmoslabs** resource group.

    ![The Lab resource group is highlighted](../media/03-lab_resource_group.jpg "Select the resource group")

1. In the **cosmoslabs** blade, select the **Azure Cosmos DB** account you recently created.

    ![The Cosmos DB resource is highlighted](../media/03-cosmos_resource.jpg "Select the Cosmos DB resource")

1. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.

    ![The Data Explorer link was selected and is blade is displayed](../media/03-data_explorer_pane.jpg "Select Data Explorer")

1. In the **Data Explorer** section, expand the **ImportDatabase** database node and then expand the **FoodCollection** container node.

    ![The Container node is displayed](../media/03-collection_node.jpg "Expand the ImportDatabase node")

1. Within the **FoodCollection** node, select the **Scale and Settings** link to view the throughput for the container. Reduce the throughput to **400 RU/s**.

    ![Scale and Settings](../media/03-collection-settings.png "Reduce throughput")

1. Within the **FoodCollection** node, select the **Items** link to view a subset of the various documents in the container. Select a few of the documents and observe the properties and structure of the documents.

    ![Items is highlighted](../media/03-documents.jpg "Select Items")

    ![An Example document is displayed](../media/03-example_document.jpg "Select a document")

> If this is your final lab, follow the steps in [Removing Lab Assets](11-cleaning_up.md) to remove all lab resources.
