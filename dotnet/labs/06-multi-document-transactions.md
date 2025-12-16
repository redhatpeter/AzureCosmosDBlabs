# Authoring Azure Cosmos DB Stored Procedures for Multi-Document Transactions

In this lab, you will author and execute multiple stored procedures within your Azure Cosmos DB instance. You will explore features unique to JavaScript stored procedures such as throwing errors for transaction rollback, logging using the JavaScript console and implementing a continuation model within a bounded execution environment.

> If this is your first lab and you have not already completed the setup for the lab content see the instructions for [Account Setup](00-account_setup.md) before starting this lab.

## Author Simple Stored Procedures

You will get started in this lab by authoring simple stored procedures that implement common server-side tasks such as adding one or more items as part of a database transaction.

> **Note** During the following steps, if you are not able to edit a query or stored procedure, move away from the **Data Explorer** window and then re-open it.

### Create Simple Stored Procedure

1. In the **Azure Cosmos DB** blade in the Azure Portal, locate and select the **Data Explorer** link on the left side of the blade.

1. In the **Data Explorer** section, expand the **NutritionDatabase** database node and then expand the **FoodCollection** container node.

1. Within the **FoodCollection** node, select the **Items** link.

1. Select the **New Stored Procedure** button (two gears icon) at the top of the **Data Explorer** section.

    ![The New Stored Procedure menu item is highlighted](../media/06-new_storedprocedure.jpg "Create a new Stored Procedure")

1. In the stored procedure tab, locate the **Stored Procedure Id** field and enter the value: **greetCaller**.

1. Replace the contents of the stored procedure editor textarea with the following JavaScript code:

    ```js
    function greetCaller(name) {
        var context = getContext();
        var response = context.getResponse();
        response.setBody("Hello " + name);
    }
    ```

    ![A new stored procedure called greetCaller is displayed](../media/06-new_greet_caller_sp.jpg "Create a new stored procedure")

    > This simple stored procedure will echo the input parameter string with the text `Hello` as a prefix.

1. Select the **Save** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: `example`.

    - If there are no param fields listed, select the **Add New Param** button.

    - In the param field, use Type **String** and enter the value: `Person`.

    - Select the **Execute** button.

        ![The stored procedure parameters are populated](../media/06-execute_sp.jpg "Execute the stored procedure")

1. In the **Result** pane at the bottom of the tab, observe the results of the stored procedure's execution.

    > The output should be `"Hello Person"`.

### Create Stored Procedure with Nested Callbacks

All Azure Cosmos DB operations within a stored procedure are asynchronous and depend on JavaScript function callbacks. A **callback function** is a JavaScript function that is used as a parameter to another JavaScript function. In the context of Azure Cosmos DB, the callback function has two parameters, one for the error object in case the operation fails, and one for the created object.

1. Select the **New Stored Procedure** button at the top of the **Data Explorer** section.

1. In the stored procedure tab, locate the **Stored Procedure Id** field and enter the value: **createDocument**.

1. Replace the contents of the stored procedure editor textarea with the following JavaScript code:

    ```js
    function createDocument(doc) {
        var context = getContext();
        var container = context.getCollection();
        var accepted = container.createDocument(
            container.getSelfLink(),
            doc,
            function (err, newItem) {
                if (err) throw new Error('Error' + err.message);
                context.getResponse().setBody(newItem);
            }
        );
        if (!accepted) return;
    }
    ```

1. Review the stored procedures code. Notice inside the JavaScript callback, users can either handle the exception or throw an error. In case a callback is not provided and there is an error, the Azure Cosmos DB runtime throws an error. This stored procedures creates a new item and uses a nested callback function to return the item as the body of the response.

1. Select the **Save** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: `My Recipes`.

    - If there are no param fields listed, select the **Add New Param** button.

    - In the param field, use Type **String** and enter the value:

        ```json
        { "foodGroup": "My Recipes", "description": "Cookies" }
        ```

    - Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe the results of the stored procedure's execution.

    ![The item created from the stored procedure is displayed](../media/06-execute_sp_02.jpg "review the item created")

    > You should see a new item in your container. Azure Cosmos DB has assigned additional fields to the item such as `id` and `_etag`.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, replace the contents of the *query editor* with the following SQL query:

    ```sql
    SELECT * FROM foods WHERE foods.foodGroup = "My Recipes" AND foods.description = "Cookies"
    ```

    > This query will retrieve the item you have just created.

1. Select the **Execute Query** button in the query tab to run the query.

1. In the **Results** pane, observe the results of your query.

1. Close the **Query** tab.

### Create Stored Procedure with Logging

1. Select the **New Stored Procedure** button at the top of the **Data Explorer** section.

1. In the stored procedure tab, locate the **Stored Procedure Id** field and enter the value: **createDocumentWithLogging**.

1. Replace the contents of the stored procedure editor with the following JavaScript code:

    ```js
    function createDocumentWithLogging(doc) {
        console.log("procedural-start");
        var context = getContext();
        var container = context.getCollection();
        console.log("metadata-retrieved");
        var accepted = container.createDocument(
            container.getSelfLink(),
            doc,
            function (err, newDoc) {
                console.log("callback-started");
                if (err) throw new Error('Error' + err.message);
                context.getResponse().setBody(newDoc.id);
            }
        );
        console.log("async-doc-creation-started");
        if (!accepted) return;
        console.log("procedural-end");
    }
    ```

    > This stored procedure demonstrates **console.log** for diagnostics. The log statements track execution flow through both the main procedural code and the asynchronous callback. This helps visualize how JavaScript's asynchronous operations work in Cosmos DB stored procedures.

1. Select the **Save** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: ``My Recipes``.

    - Select the **Add New Param** button.

    - In the new field that appears, enter the value:

        ```json
        { "foodGroup": "My Recipes", "description": "Cookies" }
        ```

    - Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe the results of the stored procedure's execution.

    > You should see the unique id of a new item in your container.

1. Select the `console.log` link in the **Result** pane to view the log data for your stored procedure execution.

    > The log output reveals the execution order:
    > ```
    > procedural-start → metadata-retrieved → async-doc-creation-started → procedural-end → callback-started
    > ```
    > Notice that all the **main procedural code executes first** (synchronously), then the **callback function executes after** the item is created (asynchronously). This demonstrates JavaScript's non-blocking execution model - the `createDocument()` operation doesn't block the main code flow.

### Create Stored Procedure with Callback Functions

1. Select the **New Stored Procedure** button at the top of the **Data Explorer** section.

1. In the stored procedure tab, locate the **Stored Procedure Id** field and enter the value: **createDocumentWithFunction**.

1. Replace the contents of the *stored procedure editor* with the following JavaScript code:

    ```js
    function createDocumentWithFunction(document) {
        var context = getContext();
        var container = context.getCollection();
        if (!container.createDocument(container.getSelfLink(), document, itemCreated))
            return;
        function itemCreated(error, newItem) {
            if (error) throw new Error('Error' + error.message);
            context.getResponse().setBody(newItem);
        }
    }
    ```

    > This is the same stored procedure as you created previously but it is using a named function instead of an implicit callback function inline.

1. Select the **Save** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: `Packaged Foods`.

    - Select the **Add New Param** button.

    - In the new field that appears, enter the value:

        ```json
        { "foodGroup": "My Recipes" }
        ```

    - Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe that the stored procedure execution has failed.

    > **Critical Concept: Partition Key Boundaries in Stored Procedures**
    > 
    > The execution failed with an error similar to:
    > ```
    > "Requests originating from scripts cannot reference partition keys other than 
    > the one for which the client request was submitted."
    > ```
    > 
    > **Why it failed:**
    > - Stored procedure was executed in the `"Packaged Foods"` partition (specified in execution parameters)
    > - The code tried to create a document with `"foodGroup": "My Recipes"` (different partition!)
    > - **Stored procedures are bound to a single partition** at execution time
    > 
    > **The Rule:** Within a stored procedure, you can ONLY:
    > - ✅ Create, read, update, or delete items in the **same partition** as the execution context
    > - ❌ Access items in **any other partition**
    > 
    > This constraint exists because:
    > - All operations in a stored procedure are part of an **ACID transaction**
    > - Transactions can only span items within the **same logical partition**
    > - This enables Cosmos DB's horizontal scaling (partitions are independent and distributed)

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    1. In the **Partition key value** section, use Type **String** and enter the value: ``Packaged Foods``.

    2. Select the **Add New Param** button.

    3. In the new field that appears, enter the value:

        ```json
        { "foodGroup": "Packaged Foods" }
        ```

    4. Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe the results of the stored procedure's execution.

    > **Success!** This time the stored procedure executed successfully because:
    > - Execution partition: `"Packaged Foods"` (specified in parameters)
    > - Document partition key: `"foodGroup": "Packaged Foods"` (matches!)
    > - Both the execution context and the document being created are in the **same partition**
    > 
    > Azure Cosmos DB assigned additional fields like `id` and `_etag` to the newly created item. This demonstrates the correct pattern for stored procedure operations - all data manipulation must occur within the partition boundary specified at execution time.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, replace the contents of the query editor with the following SQL query:

    ```sql
    SELECT * FROM foods WHERE foods.foodGroup = "Packaged Foods"
    ```

    > This query will retrieve the item you have just created.

1. Select the **Execute Query** button in the query tab to run the query.

1. In the **Results** pane, observe the results of your query.

1. Close the **Query** tab.

### Create Stored Procedure with Error Handling

1. Select the **New Stored Procedure** button at the top of the **Data Explorer** section.

1. In the stored procedure tab, locate the **Stored Procedure Id** field and enter the value: **createTwoDocuments**.

1. Replace the contents of the stored procedure editor with the following JavaScript code:

    ```js
    function createTwoDocuments(foodGroupName, foodDescription, mealName) {
        var context = getContext();
        var container = context.getCollection();
        var firstItem = {
            foodGroup: foodGroupName,
            description: foodDescription
        };
        var secondItem = {
            foodGroup: foodGroupName,
            eaten: {
                meal: mealName
            }
        };
        var firstAccepted = container.createDocument(container.getSelfLink(), firstItem,
            function (firstError, newFirstItem) {
                if (firstError) throw new Error('Error' + firstError.message);
                var secondAccepted = container.createDocument(container.getSelfLink(), secondItem,
                    function (secondError, newSecondItem) {
                        if (secondError) throw new Error('Error' + secondError.message);
                        context.getResponse().setBody({
                            foodRecord: newFirstItem,
                            mealRecord: newSecondItem
                        });
                    }
                );
                if (!secondAccepted) return;
            }
        );
        if (!firstAccepted) return;
    }
    ```

    > This stored procedure creates **two related items** using nested callbacks. Both items use the **same partition key** (`foodGroupName`), ensuring they can be created within a single transaction. This pattern is useful when your data model splits related information across multiple documents that need to be created atomically (all-or-nothing).

1. Select the **Save** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: `Vitamins`.

    - Select the **Add New Param** button two times.

    - In the first field that appears, enter the value: `Vitamins`.

    - In the second field that appears, enter the value: `Calcium`.

    - In the third field that appears, enter the value: `Breakfast`.

    - Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe the results of the stored procedure's execution.

    > **Success! Both items created.** The stored procedure successfully created two separate documents because both use the same partition key value (`Vitamins`). This demonstrates a valid multi-document transaction pattern within a single partition.

1. Replace the contents of the stored procedure editor with the following JavaScript code:

    ```js
    function createTwoDocuments(foodGroupName, foodDescription, mealName) {
        var context = getContext();
        var container = context.getCollection();
        var firstItem = {
            foodGroup: foodGroupName,
            description: foodDescription
        };
        var secondItem = {
            foodGroup: foodGroupName + "_meal",
            eaten: {
                meal: mealName
            }
        };
        var firstAccepted = container.createDocument(container.getSelfLink(), firstItem,
            function (firstError, newFirstItem) {
                if (firstError) throw new Error('Error' + firstError.message);
                console.log('Created: ' + newFirstItem.id);
                var secondAccepted = container.createDocument(container.getSelfLink(), secondItem,
                    function (secondError, newSecondItem) {
                        if (secondError) throw new Error('Error' + secondError.message);
                        console.log('Created: ' + newSecondItem.id);
                        context.getResponse().setBody({
                            foodRecord: newFirstItem,
                            mealRecord: newSecondItem
                        });
                    }
                );
                if (!secondAccepted) return;
            }
        );
        if (!firstAccepted) return;
    }
    ```

    > **Understanding Automatic Transaction Rollback**
    > 
    > Notice the key change in this version:
    > ```javascript
    > var secondItem = {
    >     foodGroup: foodGroupName + "_meal",  // Different partition key!
    >     eaten: { meal: mealName }
    > };
    > ```
    > 
    > **What will happen:**
    > - First item will be created successfully with `foodGroup: "Junk Food"`
    > - Second item attempts to use `foodGroup: "Junk Food_meal"` (different partition!)
    > - This violates the partition boundary constraint
    > - An exception is thrown
    > - **Cosmos DB automatically rolls back the entire transaction**
    > 
    > **Key Concepts:**
    > - Transactions are deeply integrated into Cosmos DB's JavaScript runtime
    > - All operations in a stored procedure are automatically wrapped in a **single ACID transaction**
    > - If the JavaScript completes **without exception**, operations are **committed**
    > - If **any exception** is thrown, the **entire transaction is rolled back**
    > - This ensures atomicity: either all operations succeed, or none do

We are going to test this transaction rollback behavior. The stored procedure will attempt to create two items with different partition keys, causing a failure that triggers automatic rollback.

1. Select the **Update** button at the top of the tab.

1. Select the **Execute** button at the top of the tab.

1. In the **Input parameters** popup that appears, perform the following actions:

    - In the **Partition key value** section, use Type **String** and enter the value: `Junk Food`.

    - Select the **Add New Param** button until three param fields are listed.

    - In the first field that appears, enter the value: `Junk Food`.

    - In the second field that appears, enter the value: `Chips`.

    - In the third field that appears, enter the value: `Midnight Snack`.

    - Select the **Execute** button.

1. In the **Result** pane at the bottom of the tab, observe that the stored procedure execution has failed.

    > **Transaction Rollback Confirmed**
    > 
    > The stored procedure failed because:
    > 1. First item (`"Junk Food"`) was successfully created
    > 2. Second item attempted to use `"Junk Food_meal"` as partition key
    > 3. Exception was thrown due to partition boundary violation
    > 4. **Both items were rolled back** - neither exists in the database
    > 
    > Even though the first `console.log('Created: ' + newFirstItem.id)` would have executed, the entire transaction was undone. This is the power of automatic transaction management in Cosmos DB stored procedures.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, replace the contents of the query editor with the following SQL query:

    ```sql
    SELECT * FROM foods WHERE foods.foodGroup = "Junk Food"
    ```

    > **Verifying the Rollback:** This query searches for items with `"foodGroup": "Junk Food"`. If the transaction rollback worked correctly, you should see an **empty result set** `[]` because:
    > - The first item was created during the stored procedure execution
    > - But when the second item failed, the entire transaction was rolled back
    > - Neither the first nor second item persisted in the database
    > 
    > This proves that stored procedures provide true ACID transaction guarantees within a single partition.

1. Select the **Execute Query** button in the query tab to run the query. You should see only an empty array.

1. In the **Results** pane, observe the results of your query.

1. Close the **Query** tab.

> If this is your final lab, follow the steps in [Removing Lab Assets](11-cleaning_up.md) to remove all lab resources.
