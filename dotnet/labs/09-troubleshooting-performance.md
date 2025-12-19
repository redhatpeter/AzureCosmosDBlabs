# Troubleshooting Azure Cosmos DB Performance

In this lab, you will use the .NET SDK to tune Azure Cosmos DB requests to optimize the performance and cost of your application.

> If this is your first lab and you have not already completed the setup for the lab content see the instructions for [Account Setup](00-account_setup.md) before starting this lab.

## Create the FinancialDatabase and Containers

Before starting this lab, you need to create the database and containers for the financial data.

1. In the Azure Portal, navigate to your Azure Cosmos DB account.

2. From within the **Azure Cosmos DB** blade, select the **Data Explorer** tab on the left.

3. At the top of the **Data Explorer** section, select **New Database**.

4. In the **New Database** pane on the right, enter the following values:

   - **Database id**: Enter `FinancialDatabase`
   - **Throughput**: Select **Manual** and leave at default

5. Select **OK** to create the database.

6. Now create the first container. Expand the **FinancialDatabase** database, then select the **...** (more options) button next to it.

7. Select **New Container**.

8. In the **New Container** pane on the right, enter the following values:

   - **Database id**: Select **Use existing** and choose `FinancialDatabase`
   - **Container id**: Enter `PeopleCollection`
   - **Partition key**: Enter `/accountHolder/LastName`
   - **Throughput**: Select **Manual** and enter `400` RU/s

9. Select **OK** to create the container.

   > **Note**: The partition key `/accountHolder/LastName` is used because the Member objects store a Person object in the `accountHolder` property, and Person objects from the Bogus library have a `LastName` property.

10. Create the second container. Select the **...** (more options) button next to **FinancialDatabase** again.

11. Select **New Container** and enter the following values:

    - **Database id**: Select **Use existing** and choose `FinancialDatabase`
    - **Container id**: Enter `TransactionCollection`
    - **Partition key**: Enter `/costCenter`
    - **Throughput**: **IMPORTANT** - Select **Manual** (NOT Autoscale) and enter `400` RU/s

12. Select **OK** to create the container.

    > **Critical Note**: You must create `TransactionCollection` with **Manual** throughput set to exactly **400 RU/s**. This low throughput setting is required for the "Troubleshooting Requests" section later in this lab, where you will observe throttling behavior (HTTP 429 errors). If you accidentally create the container with Autoscale or higher throughput, you will need to delete and recreate it with Manual 400 RU/s before proceeding with the throttling exercises.

## Create a .NET Core Project

1. On your local machine, locate the CosmosLabs folder in your Documents folder and open the `Lab09` folder that will be used to contain the content of your .NET Core project.

1. In the `Lab09` folder, right-click the folder and select the **Open with Code** menu option.

    > Alternatively, you can run a terminal in your current directory and execute the ``code .`` command.

1. In the Visual Studio Code window that appears, right-click the **Explorer** pane and select the **Open in Terminal** menu option.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet restore
    ```

    > This command will restore all packages specified as dependencies in the project.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet build
    ```

    > This command will build the project.

1. In the **Explorer** pane, select the **DataTypes.cs**
1. Review the file, notice it contains the data classes you will be working with in the following steps.

1. Select the **Program.cs** link in the **Explorer** pane to open the file in the editor.

1. For the ``_endpointUri`` variable, replace the placeholder value with the **URI** value and for the ``_primaryKey`` variable, replace the placeholder value with the **PRIMARY KEY** value from your Azure Cosmos DB account. Use [these instructions](00-account_setup.md) to get these values if you do not already have them:

    - For example, if your **uri** is ``https://cosmosacct.documents.azure.com:443/``, your new variable assignment will look like this: 

    ```csharp
    private static readonly string _endpointUri = "https://cosmosacct.documents.azure.com:443/";
    ```

    - For example, if your **primary key** is ``elzirrKCnXlacvh1CRAnQdYVbVLspmYHQyYrhx0PltHi8wn5lHVHFnd1Xm3ad5cn4TUcH4U0MSeHsVykkFPHpQ==``, your new variable assignment will look like this:

    ```csharp
    private static readonly string _primaryKey = "elzirrKCnXlacvh1CRAnQdYVbVLspmYHQyYrhx0PltHi8wn5lHVHFnd1Xm3ad5cn4TUcH4U0MSeHsVykkFPHpQ==";
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet build
    ```

## Examining Response Headers

Azure Cosmos DB returns various response headers that can give you more metadata about your request and what operations occurred on the server-side. The .NET SDK exposes many of these headers to you as properties of the `ResourceResponse<>` class.

### Observe RU Charge for Large Item

1. Locate the following code within the `Main` method:

    ```csharp

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

    ```

1. After the last line of code, add a new line of code to call a new function we will create in the next step:

    ```csharp
    await CreateMember(peopleContainer);
    ```

1. Next create a new function that creates a new object and stores it in a variable named `member`:

    ```csharp
    private static async Task CreateMember(Container peopleContainer)
    {
        object member = new Member { accountHolder = new Bogus.Person() };

    }
    ```

    > The **Bogus** library has a special helper class (`Bogus.Person`) that will generate a fictional person with randomized properties. Here's an example of a fictional person JSON document:

    ```js
    {
        "Gender": 1,
        "FirstName": "Rosalie",
        "LastName": "Dach",
        "FullName": "Rosalie Dach",
        "UserName": "Rosalie_Dach",
        "Avatar": "https://s3.amazonaws.com/uifaces/faces/twitter/mastermindesign/128.jpg",
        "Email": "Rosalie27@gmail.com",
        "DateOfBirth": "1962-02-22T21:48:51.9514906-05:00",
        "Address": {
            "Street": "79569 Wilton Trail",
            "Suite": "Suite 183",
            "City": "Caramouth",
            "ZipCode": "85941-7829",
            "Geo": {
                "Lat": -62.1607,
                "Lng": -123.9278
            }
        },
        "Phone": "303.318.0433 x5168",
        "Website": "gerhard.com",
        "Company": {
            "Name": "Mertz - Gibson",
            "CatchPhrase": "Focused even-keeled policy",
            "Bs": "architect mission-critical markets"
        }
    }
    ```

1. Add a new line of code to invoke the **CreateItemAsync** method of the **Container** instance using the **member** variable as a parameter:

    ```csharp
    ItemResponse<object> response = await peopleContainer.CreateItemAsync(member);
    ```

1. After the last line of code in the using block, add a new line of code to print out the value of the **RequestCharge** property of the **ItemResponse<>** instance and return the RequestCharge (we will use this value in a later exercise):

    ```csharp
    await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
    return response.RequestCharge;
    ```

1. The `Main` and `CreateMember` methods should now look like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        await CreateMember(peopleContainer);
    }

    private static async Task<double> CreateMember(Container peopleContainer)
    {
        object member = new Member { accountHolder = new Bogus.Person() };
        ItemResponse<object> response = await peopleContainer.CreateItemAsync(member);
        await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
        return response.RequestCharge;
    }
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the results of the console project. You should see the document creation operation use approximately `15  RU/s`.

1. Return to the **Azure Portal** (<http://portal.azure.com>).

1. On the left side of the portal, select the **Resource groups** link.

1. In the **Resource groups** blade, locate and select the **cosmoslab** resource group.

1. In the **cosmoslab** blade, select the **Azure Cosmos DB** account you recently created.

1. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node and then select the **PeopleCollection** node.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, notice the following SQL query.

    ```sql
    SELECT * FROM c
    ```

1. Select the **Execute Query** button in the query tab to run the query.

1. In the **Results** pane, observe the results of your query. Click **Query Stats** you should see an execution cost of ~3 RU/s.

1. Return to the currently open **Visual Studio Code** editor containing your .NET Core project.

1. In the Visual Studio Code window, select the **Program.cs** file to open an editor tab for the file.

1. To view the RU charge for inserting a very large document, we will use the **Bogus** library to create a fictional family on our Member object. To create a fictional family, we will generate a spouse and an array of 4 fictional children:

    ```js
    {
        "accountHolder":  { ... },
        "relatives": {
            "spouse": { ... },
            "children": [
                { ... },
                { ... },
                { ... },
                { ... }
            ]
        }
    }
    ```

    Each property will have a **Bogus**-generated fictional person. This should create a large JSON document that we can use to observe RU charges.

1. Within the **Program.cs** editor tab, locate the `CreateMember` method.

1. Within the `CreateMember` method, locate the following line of code:

    ```csharp
    object member = new Member { accountHolder = new Bogus.Person() };
    ```

    Replace that line of code with the following code:

    ```csharp
    object member = new Member
    {
        accountHolder = new Bogus.Person(),
        relatives = new Family
        {
            spouse = new Bogus.Person(),
            children = Enumerable.Range(0, 4).Select(r => new Bogus.Person())
        }
    };
    ```

    > This new block of code will create the large JSON object discussed above.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the results of the console project. You should see this new operation require far more  RU/s than the simple JSON document at ~50 RU/s (last one was ~15 RU/s).

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node and then select the **PeopleCollection** node.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, replace the contents of the *query editor* with the following SQL query. This query will return the only item in your container with a property named **Children**:

    ```sql
    SELECT * FROM c WHERE IS_DEFINED(c.relatives)
    ```

1. Select the **Execute Query** button in the query tab to run the query.

1. In the **Results** pane, observe the results of your query, you should see more data returned and a slightly higher RU cost.

### Tune Index Policy

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node, expand the **PeopleCollection** node, and then select the **Scale & Settings** option.

1. In the **Settings** section, locate the **Indexing Policy** field and observe the current default indexing policy:

    ```js
    {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [
            {
                "path": "/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/\"_etag\"/?"
            }
        ],
        "spatialIndexes": [
            {
                "path": "/*",
                "types": [
                    "Point",
                    "LineString",
                    "Polygon",
                    "MultiPolygon"
                ]
            }
        ]
    }
    ```

    > This policy will index all paths in your JSON document, except for _etag which is never used in queries. This policy will also index spatial data.

1. Replace the indexing policy with a new policy. This new policy will exclude the `/relatives/*` path from indexing effectively removing the **Children** property of your large JSON document from the index:

    ```js
    {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [
            {
                "path": "/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/\"_etag\"/?"
            },
            {
                "path":"/relatives/*"
            }
        ],
        "spatialIndexes": [
            {
                "path": "/*",
                "types": [
                    "Point",
                    "LineString",
                    "Polygon",
                    "MultiPolygon"
                ]
            }
        ]
    }
    ```

1. Select the **Save** button at the top of the section to persist your new indexing policy.

1. Select the **New SQL Query** button at the top of the **Data Explorer** section.

1. In the query tab, replace the contents of the query editor with the following SQL query:

    ```sql
    SELECT * FROM c WHERE IS_DEFINED(c.relatives)
    ```

1. Select the **Execute Query** button in the query tab to run the query.

    > You will see immediately that you can still determine if the **/relatives** path is defined.

1. In the query tab, replace the contents of the query editor with the following SQL query:

    ```sql
    SELECT * FROM c WHERE IS_DEFINED(c.relatives) ORDER BY c.relatives.Spouse.FirstName
    ```

1. Select the **Execute Query** button in the query tab to run the query.

    > This query will fail immediately because the `/relatives/Spouse/FirstName` path has been excluded from the index. Azure Cosmos DB requires properties used in ORDER BY clauses to be indexed. Remember: when you exclude a path from indexing, you cannot use that property in query operations that require an index, such as ORDER BY, filtering with range operators, or certain JOIN conditions.

1. Return to the currently open **Visual Studio Code** editor containing your .NET Core project.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the results of the console project. You should see a significant reduction in RU/s consumption (~26 RU/s compared to ~48 RU/s previously) required to create this item. This represents approximately a **48% reduction** in write costs. The savings occur because Azure Cosmos DB no longer needs to index the `/relatives/*` path, which contains the large nested objects (spouse and 4 children). By excluding this path from the index, you've reduced the index maintenance overhead during write operations while still maintaining the ability to query for the existence of the `relatives` property using `IS_DEFINED()`.

## Troubleshooting Requests

In this section, you will use the .NET SDK to intentionally exceed the provisioned throughput capacity of a container to observe throttling behavior. Azure Cosmos DB evaluates Request Unit (RU) consumption on a **per-second basis**. When your application sends requests that consume more RU/s than the provisioned amount, Azure Cosmos DB **rate-limits** (throttles) those requests to protect the service and ensure fair resource allocation.

**How Throttling Works:**

When rate-limiting occurs, Azure Cosmos DB:
1. **Rejects the request immediately** with an HTTP status code of **`429 RequestRateTooLargeException`**
2. **Returns the `x-ms-retry-after-ms` header**, indicating how many milliseconds the client should wait before retrying
3. **Continues throttling** until the request rate drops below the provisioned throughput level

**Real-World Example:**

Imagine your `TransactionCollection` is provisioned with **400 RU/s**. If your application attempts to create 5,000 items simultaneously in parallel:
- Each item creation might cost ~5 RU
- Total demand: ~25,000 RU in 1-2 seconds
- **Result**: Many requests will receive HTTP 429 errors because you're consuming RU/s far faster than the 400 RU/s capacity

In the following exercises, you will intentionally trigger this throttling behavior by creating thousands of items in parallel against a container with only 400 RU/s provisioned. This will help you understand how to identify and handle throttling in production applications.

### Verify R/U Throughput for TransactionCollection

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node, expand the **TransactionCollection** node, and then select the **Scale & Settings** option.

1. In the **Settings** section, verify that the **Throughput** is set to **Manual** mode with **400 RU/s**.

    > **Note**: If you see the container is set to **Autoscale** or has a higher throughput, you will need to delete and recreate the `TransactionCollection` with Manual 400 RU/s as specified in the "Create the FinancialDatabase and Containers" section at the beginning of this lab. The 400 RU/s setting is the minimum throughput for a container and is necessary to observe throttling behavior in the following exercises.

### Observing Throttling (HTTP 429)

1. Return to the currently open **Visual Studio Code** editor containing your .NET Core project.

1. Select the **Program.cs** link in the **Explorer** pane to open the file in the editor.

1. Locate the `await CreateMember(peopleContainer)` line within the `Main` method. Comment out this line and add a new line below so it looks like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        await CreateTransactions(transactionContainer);
    }
    ```

1. Below the `CreateMember` method create a new method `CreateTransactions`:

    ```csharp
    private static async Task CreateTransactions(Container transactionContainer)
    {

    }
    ```

1. Add the following code to create a collection of `Transaction` instances:

    ```csharp
    var transactions = new Bogus.Faker<Transaction>()
        .RuleFor(t => t.id, (fake) => Guid.NewGuid().ToString())
        .RuleFor(t => t.amount, (fake) => Math.Round(fake.Random.Double(5, 500), 2))
        .RuleFor(t => t.processed, (fake) => fake.Random.Bool(0.6f))
        .RuleFor(t => t.paidBy, (fake) => $"{fake.Name.FirstName().ToLower()}.{fake.Name.LastName().ToLower()}")
        .RuleFor(t => t.costCenter, (fake) => fake.Commerce.Department(1).ToLower())
        .GenerateLazy(100);
    ```

1. Add the following foreach block to iterate over the `Transaction` instances:

    ```csharp
    foreach(var transaction in transactions)
    {

    }
    ```

1. Within the `foreach` block, add the following line of code to asynchronously create an item and save the result of the creation task to a variable:

    ```csharp
    ItemResponse<Transaction> result = await transactionContainer.CreateItemAsync(transaction);
    ```

    > The `CreateItemAsync` method of the `Container` class takes in an object that you would like to serialize into JSON and store as an item within the specified collection.

1. Still within the `foreach` block, add the following line of code to write the value of the newly created resource's `id` property to the console:

    ```csharp
    await Console.Out.WriteLineAsync($"Item Created\t{result.Resource.id}");
    ```

    > The `ItemResponse` type has a property named `Resource` that can give you access to the item instance resulting from the operation.

1. Your `CreateTransactions` method should look like this:

    ```csharp
    private static async Task CreateTransactions(Container transactionContainer)
    {
        var transactions = new Bogus.Faker<Transaction>()
            .RuleFor(t => t.id, (fake) => Guid.NewGuid().ToString())
            .RuleFor(t => t.amount, (fake) => Math.Round(fake.Random.Double(5, 500), 2))
            .RuleFor(t => t.processed, (fake) => fake.Random.Bool(0.6f))
            .RuleFor(t => t.paidBy, (fake) => $"{fake.Name.FirstName().ToLower()}.{fake.Name.LastName().ToLower()}")
            .RuleFor(t => t.costCenter, (fake) => fake.Commerce.Department(1).ToLower())
            .GenerateLazy(100);

        foreach(var transaction in transactions)
        {
            ItemResponse<Transaction> result = await transactionContainer.CreateItemAsync(transaction);
            await Console.Out.WriteLineAsync($"Item Created\t{result.Resource.id}");
        }
    }
    ```

    > As a reminder, the Bogus library generates a set of test data. In this example, you are creating 100 items using the Bogus library and the rules listed above. The **GenerateLazy** method tells the Bogus library to prepare for a request of 100 items by returning a variable of type `IEnumerable<Transaction>`. Since LINQ uses deferred execution by default, the items aren't actually created until the collection is iterated. The **foreach** loop at the end of this code block iterates over the collection and creates items in Azure Cosmos DB.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application. You should see a list of item ids associated with new items that are being created by this tool.

1. Back in the code editor tab, locate the following lines of code:

    ```csharp
    foreach (var transaction in transactions)
    {
        ItemResponse<Transaction> result = await transactionContainer.CreateItemAsync(transaction);
        await Console.Out.WriteLineAsync($"Item Created\t{result.Resource.id}");
    }
    ```

    Replace those lines of code with the following code:

    ```csharp
    List<Task<ItemResponse<Transaction>>> tasks = new List<Task<ItemResponse<Transaction>>>();
    foreach (var transaction in transactions)
    {
        Task<ItemResponse<Transaction>> resultTask = transactionContainer.CreateItemAsync(transaction);
        tasks.Add(resultTask);
    }
    Task.WaitAll(tasks.ToArray());
    foreach (var task in tasks)
    {
        await Console.Out.WriteLineAsync($"Item Created\t{task.Result.Resource.id}");
    }
    ```

    > We are going to attempt to run as many of these creation tasks in parallel as possible. Remember, our container is configured at the minimum of 400 RU/s.

1. Your `CreateTransactions` method should look like this:

    ```csharp
    private static async Task CreateTransactions(Container transactionContainer)
    {
        var transactions = new Bogus.Faker<Transaction>()
            .RuleFor(t => t.id, (fake) => Guid.NewGuid().ToString())
            .RuleFor(t => t.amount, (fake) => Math.Round(fake.Random.Double(5, 500), 2))
            .RuleFor(t => t.processed, (fake) => fake.Random.Bool(0.6f))
            .RuleFor(t => t.paidBy, (fake) => $"{fake.Name.FirstName().ToLower()}.{fake.Name.LastName().ToLower()}")
            .RuleFor(t => t.costCenter, (fake) => fake.Commerce.Department(1).ToLower())
            .GenerateLazy(100);

        List<Task<ItemResponse<Transaction>>> tasks = new List<Task<ItemResponse<Transaction>>>();

        foreach (var transaction in transactions)
        {
            Task<ItemResponse<Transaction>> resultTask = transactionContainer.CreateItemAsync(transaction);
            tasks.Add(resultTask);
        }

        Task.WaitAll(tasks.ToArray());

        foreach (var task in tasks)
        {
            await Console.Out.WriteLineAsync($"Item Created\t{task.Result.Resource.id}");
        }
    }
    ```

    - The first **foreach** loops iterates over the created transactions and creates asynchronous tasks which are stored in an `List`. Each asynchronous task will issue a request to Azure Cosmos DB. These requests are issued in parallel and could generate a `429 - too many requests` exception since your container does not have enough throughput provisioned to handle the volume of requests.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > This query should execute successfully. We are only creating 100 items and we most likely will not run into any throughput issues here.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    .GenerateLazy(100);
    ```

    Replace that line of code with the following code:

    ```csharp
    .GenerateLazy(5000);
    ```

    > We are going to try and create 5000 items in parallel to see if we can hit out throughput limit.

1. **Save** all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe that the application will crash after some time.

    > This query will most likely hit our throughput limit. You will see multiple error messages indicating that specific requests have failed.

### Increasing R/U Throughput to Reduce Throttling

1. Switch back to the Azure Portal, in the **Data Explorer** section, expand the **FinancialDatabase** database node, expand the **TransactionCollection** node, and then select the **Scale & Settings** option.

1. In the **Settings** section, locate the **Throughput** field and update it's value to **10000**.

1. Select the **Save** button at the top of the section to persist your new throughput allocation.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe that the application will complete after some time.

1. Return to the **Settings** section in the **Azure Portal** and change the **Throughput** value back to **400**.

1. Select the **Save** button at the top of the section to persist your new throughput allocation.

## Tuning Queries and Reads

You will now tune your requests to Azure Cosmos DB by manipulating the SQL query and properties of the **RequestOptions** class in the .NET SDK.

### Measuring RU Charge

1. Locate the `Main` method to comment out `CreateTransactions` and add a new line `await QueryTransactions(transactionContainer);`. The method should look like this:

    ```csharp
    public static async Task Main(string[] args)
    {
        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        //await CreateTransactions(transactionContainer);
        await QueryTransactions(transactionContainer);
    }
    ```

1. Create a new function `QueryTransactions` and add the following line of code that will store a SQL query in a string variable:

    ```csharp
    private static async Task QueryTransactions(Container transactionContainer)
    {
        string sql = "SELECT TOP 1000 * FROM c WHERE c.processed = true ORDER BY c.amount DESC";

    }
    ```

    > This query will perform a cross-partition ORDER BY and only return the top 1000 out of 50000 items.

1. Add the following line of code to create a item query instance:

    ```csharp
    FeedIterator<Transaction> query = transactionContainer.GetItemQueryIterator<Transaction>(sql);
    ```

1. Add the following line of code to get the first "page" of results:

    ```csharp
    var result = await query.ReadNextAsync();
    ```

    > We will not enumerate the full result set. We are only interested in the metrics for the first page of results.

1. Add the following lines of code to print out the Request Charge metric for the query to the console:

    ```csharp
    await Console.Out.WriteLineAsync($"Request Charge: {result.RequestCharge} RU/s");
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application. You should see the **Request Charge** metric printed out in your console window. It should be ~81 RU/s.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    string sql = "SELECT TOP 1000 * FROM c WHERE c.processed = true ORDER BY c.amount DESC";
    ```

    Replace that line of code with the following code:

    ```csharp
    string sql = "SELECT * FROM c WHERE c.processed = true";
    ```

    > This new query does not perform a cross-partition ORDER BY.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application. You should see a slight reduction in both the **Request Charge** value to ~35 RU/s

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    string sql = "SELECT * FROM c WHERE c.processed = true";
    ```

    Replace that line of code with the following code:

    ```csharp
    string sql = "SELECT * FROM c";
    ```

    > This new query does not filter the result set.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application. This query should be ~21 RU/s. Observe the slight differences in the various metric values from last few queries.

1. Back in the code editor tab, locate the following line of code:

     ```csharp
     string sql = "SELECT * FROM c";
     ```

     Replace that line of code with the following code:

     ```csharp
     string sql = "SELECT c.id FROM c";
     ```

     > **Understanding the RU difference between `SELECT * FROM c` and `SELECT c.id FROM c`:**
     > 
     > While both queries perform a full container scan without filtering, they differ slightly in cost:
     > 
     > - **`SELECT * FROM c`**: Returns entire documents with all properties. The query engine reads and transfers complete items.
     > 
     > - **`SELECT c.id FROM c`**: Returns only the `id` property from each document. Although this requires property projection (extracting specific fields), the **reduced data transfer size** results in slightly lower RU consumption.
     > 
     > **Key Insight**: For queries without filters or ORDER BY clauses, selecting fewer properties typically reduces RU charges because the savings from transferring less data over the network outweigh any projection overhead. However, the difference is minimal (typically less than 1 RU) for simple queries on small result sets. The real RU savings from projection become more significant with larger result sets or when selecting a small subset of properties from documents with many fields.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application. You should see the RU charge is very similar to `SELECT *` (~21-22 RU/s range), with potentially a slight reduction due to the smaller payload being transferred.

### Managing SDK Query Options

1. Locate the `CreateTransactions2` method and delete the code added for the previous section so it again looks like this:

    ```csharp
    private static async Task QueryTransactions2(Container transactionContainer)
    {

    }
    ```

1. Add the following lines of code to create variables to configure query options:

    ```csharp
    int maxItemCount = 100;
    int maxDegreeOfParallelism = 1;
    int maxBufferedItemCount = 0;
    ```

1. Add the following lines of code to configure options for a query from the variables:

    ```csharp
    QueryRequestOptions options = new QueryRequestOptions
    {
        MaxItemCount = maxItemCount,
        MaxBufferedItemCount = maxBufferedItemCount,
        MaxConcurrency = maxDegreeOfParallelism
    };
    ```

1. Add the following lines of code to write various values to the console window:

    ```csharp
    await Console.Out.WriteLineAsync($"MaxItemCount:\t{maxItemCount}");
    await Console.Out.WriteLineAsync($"MaxDegreeOfParallelism:\t{maxDegreeOfParallelism}");
    await Console.Out.WriteLineAsync($"MaxBufferedItemCount:\t{maxBufferedItemCount}");
    ```

1. Add the following line of code that will store a SQL query in a string variable:

    ```csharp
    string sql = "SELECT * FROM c WHERE c.processed = true ORDER BY c.amount DESC";
    ```

    > This query will perform a cross-partition ORDER BY on a filtered result set.

1. Add the following line of code to create and start new a high-precision timer:

    ```csharp
    Stopwatch timer = Stopwatch.StartNew();
    ```

1. Add the following line of code to create a item query instance:

    ```csharp
    FeedIterator<Transaction> query = transactionContainer.GetItemQueryIterator<Transaction>(sql, requestOptions: options);
    ```

1. Add the following lines of code to enumerate the result set.

    ```csharp
    while (query.HasMoreResults)  
    {
        var result = await query.ReadNextAsync();
    }
    ```

    > Since the results are paged, we will need to call the `ReadNextAsync` method multiple times in a while loop.

1. Add the following line of code stop the timer:

    ```csharp
    timer.Stop();
    ```

1. Add the following line of code to write the timer's results to the console window:

    ```csharp
    await Console.Out.WriteLineAsync($"Elapsed Time:\t{timer.Elapsed.TotalSeconds}");
    ```

1. The `QueryTransactions` method should now look like this:

    ```csharp
    private static async Task QueryTransactions2(Container transactionContainer)
    {
        int maxItemCount = 100;
        int maxDegreeOfParallelism = 1;
        int maxBufferedItemCount = 0;

        QueryRequestOptions options = new QueryRequestOptions
        {
            MaxItemCount = maxItemCount,
            MaxBufferedItemCount = maxBufferedItemCount,
            MaxConcurrency = maxDegreeOfParallelism
        };

        await Console.Out.WriteLineAsync($"MaxItemCount:\t{maxItemCount}");
        await Console.Out.WriteLineAsync($"MaxDegreeOfParallelism:\t{maxDegreeOfParallelism}");
        await Console.Out.WriteLineAsync($"MaxBufferedItemCount:\t{maxBufferedItemCount}");

        string sql = "SELECT * FROM c WHERE c.processed = true ORDER BY c.amount DESC";

        Stopwatch timer = Stopwatch.StartNew();

        FeedIterator<Transaction> query = transactionContainer.GetItemQueryIterator<Transaction>(sql, requestOptions: options);
        while (query.HasMoreResults)  
        {
            var result = await query.ReadNextAsync();
        }
        timer.Stop();
        await Console.Out.WriteLineAsync($"Elapsed Time:\t{timer.Elapsed.TotalSeconds}");
    }
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > This initial query should take an unexpectedly long amount of time. This will require us to optimize our SDK options.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxDegreeOfParallelism = 1;
    ```

    Replace that line of code with the following:

    ```csharp
    int maxDegreeOfParallelism = 5;
    ```

    > Setting the `maxDegreeOfParallelism` query parameter to a value of `1` effectively eliminates parallelism. Here we "bump up" the parallelism to a value of `5`.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > You should see a very slight positive impact considering you now have some form of parallelism.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxBufferedItemCount = 0;
    ```

    Replace that line of code with the following code:

    ```csharp
    int maxBufferedItemCount = -1;
    ```

    > Setting the `MaxBufferedItemCount` property to a value of `-1` effectively tells the SDK to manage this setting.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > Again, this should have a slight positive impact on your performance time.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxDegreeOfParallelism = 5;
    ```

    Replace that line of code with the following code:

    ```csharp
    int maxDegreeOfParallelism = -1;
    ```

    > **Understanding MaxDegreeOfParallelism optimization:**
    > 
    > Setting `MaxDegreeOfParallelism` to `-1` allows the Azure Cosmos DB SDK to automatically determine the optimal degree of parallelism based on your container's partition count. This is particularly beneficial for cross-partition queries like ORDER BY operations.
    > 
    > **How Parallel Query Works:**
    > - The SDK queries multiple partitions **simultaneously** to improve throughput
    > - Data from each individual partition is still fetched **serially** (one page at a time per partition)
    > - The SDK coordinates results across partitions to satisfy the query requirements
    > 
    > **Performance Impact:**
    > Based on your test results, you may observe that the performance improvement from `MaxDegreeOfParallelism = 5` (22.03s) to `MaxDegreeOfParallelism = -1` (21.97s) is minimal. This is because:
    > - Your container may have a similar number of partitions (~5)
    > - Network latency and client-side processing become the bottleneck
    > - The query is already well-optimized with `MaxBufferedItemCount = -1`
    > 
    > **Best Practice:** Use `-1` to let the SDK automatically scale parallelism as your container grows and partitions increase over time, ensuring optimal performance without code changes.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > You should see execution time similar to the previous run (~21-22 seconds). The performance is comparable because the SDK was already using near-optimal parallelism at `5`, and auto-tuning to `-1` provides flexibility for future partition growth rather than immediate dramatic improvement.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxItemCount = 100;
    ```

    Replace that line of code with the following code:

    ```csharp
    int maxItemCount = 500;
    ```

    > We are increasing the amount of items returned per "page" in an attempt to improve the performance of the query.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > You will notice that the query performance improved **dramatically** from ~22 seconds down to ~8 seconds. This represents a **64% reduction** in query execution time simply by increasing the page size from 100 to 500 items. The significant improvement indicates that the query was bottlenecked by network round-trips between the client and Cosmos DB. With larger page sizes, fewer round-trips are needed to retrieve all results.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxItemCount = 500;
    ```

    Replace that line of code with the following code:

    ```csharp
    int maxItemCount = 1000;
    ```

    > For large queries, increasing the page size to 1,000 items can further reduce network round-trips and improve performance.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > The query execution time dropped to approximately **7.76 seconds**, representing another modest improvement (3% faster than 500 items/page, 65% faster than the original 100 items/page baseline). The diminishing returns indicate we're approaching an optimal balance where fewer round-trips meet practical limits.

1. Back in the code editor tab, locate the following line of code:

    ```csharp
    int maxBufferedItemCount = -1;
    ```

    Replace that line of code with the following code:

    ```csharp
    int maxBufferedItemCount = 50000;
    ```

    > Parallel query is designed to pre-fetch results while the current batch of results is being processed by the client. The pre-fetching helps in overall latency improvement of a query. **MaxBufferedItemCount** is the parameter to limit the number of pre-fetched results. Setting MaxBufferedItemCount to the expected number of results returned (or a higher number) allows the query to receive maximum benefit from pre-fetching.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > With MaxBufferedItemCount set to 50,000 and MaxItemCount at 1,000, the query execution time reached approximately **7.12 seconds**. This represents the best performance yet—a **67.6% improvement** over the original baseline of 21.97 seconds with 100 items/page.

#### Key Takeaways: Page Size Optimization

The MaxItemCount tuning revealed a dramatic performance improvement that far exceeded the gains from parallelism optimization:

**Performance Progression:**
- **MaxItemCount = 100 (baseline)**: 21.97 seconds
- **MaxItemCount = 500**: 8.04 seconds (63% faster)
- **MaxItemCount = 1000**: 7.76 seconds (65% faster)
- **MaxItemCount = 50000 + Buffer tuning**: 7.12 seconds (67.6% faster)

**Why Page Size Had Dramatic Impact (vs. Minimal Parallelism Impact):**

The 67% performance improvement from increasing MaxItemCount reveals that the **client-side round-trip overhead** was the primary bottleneck, not server-side query execution:

1. **Round-Trip Reduction**: With 100 items/page, retrieving 50,000 items requires ~500 network round-trips. At 1,000 items/page, only ~50 round-trips are needed. Each round-trip incurs network latency plus client processing overhead.

2. **Client Bottleneck Revealed**: The massive improvement shows that the client computer was spending most of its time waiting for network responses rather than processing data. In contrast, the minimal improvement from MaxDegreeOfParallelism (3-4%) indicated the server was already efficiently distributing work across the ~5 physical partitions.

3. **Best Practices for Page Size Selection**:
   - For large result sets (1000+ items), use MaxItemCount values of 1,000 or higher to minimize round-trips
   - For small result sets or UI pagination, smaller page sizes (50-100) are appropriate
   - Monitor both query execution time and RU consumption when tuning—larger pages reduce time but don't significantly impact RU charges
   - The optimal page size balances network efficiency with client memory constraints

4. **Complementary Optimizations**: Combining large page sizes with MaxBufferedItemCount allows the SDK to pre-fetch subsequent pages while processing current results, further reducing perceived latency.

### Reading and Querying Items

1. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node, expand the **PeopleCollection** node, and then select the **Items** option.

1. Take note of the **id** property value of any document as well as that document's **partition key**.

    ![The PeopleCollection is displayed with an item selected.  The id and Lastname is highlighted.](../media/09-find-id-key.jpg "Find an item and record its id property")

1. Locate the `Main` method and comment the last line and add a new line `await QueryMember(peopleContainer);` so it looks like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        //await CreateTransactions(transactionContainer);
        //await QueryTransactions(transactionContainer);
        await QueryMember(peopleContainer);
    }
    ```

1. Below the `QueryTransactions` method create a new method that looks like this:

    ```csharp
    private static async Task QueryMember(Container peopleContainer)
    {

    }
    ```

1. Add the following line of code that will store a SQL query in a string variable (replacing **example.document** with the **id** value that you noted earlier):

    ```csharp
    string sql = "SELECT TOP 1 * FROM c WHERE c.id = 'example.document'";
    ```

    > This query will find a single item matching the specified unique id.

1. Add the following line of code to create a item query instance:

    ```csharp
    FeedIterator<object> query = peopleContainer.GetItemQueryIterator<object>(sql);
    ```

1. Add the following line of code to get the first page of results and then store them in a variable of type **FeedResponse<>**:

    ```csharp
    FeedResponse<object> response = await query.ReadNextAsync();
    ```

    > We only need to retrieve a single page since we are getting the ``TOP 1`` items from the .

1. Add the following lines of code to print out the value of the **RequestCharge** property of the **FeedResponse<>** instance and then the content of the retrieved item:

    ```csharp
    await Console.Out.WriteLineAsync($"{response.Resource.First()}");
    await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
    ```

1. The method should look like this:

    ```csharp
    private static async Task QueryMember(Container peopleContainer)
    {
        string sql = "SELECT TOP 1 * FROM c WHERE c.id = '372a9e8e-da22-4f7a-aff8-3a86f31b2934'";
        FeedIterator<object> query = peopleContainer.GetItemQueryIterator<object>(sql);
        FeedResponse<object> response = await query.ReadNextAsync();

        await Console.Out.WriteLineAsync($"{response.Resource.First()}");
        await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
    }
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > You should see the amount of ~ 3 RU/s used to query for the item. Make note of the **LastName** object property value as you will use it in the next step.

1. Locate the `Main` method and comment the last line and add a new line `await ReadMember(peopleContainer);` so it looks like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        //await CreateTransactions(transactionContainer);
        //await QueryTransactions(transactionContainer);
        //await QueryMember(peopleContainer);
        await ReadMember(peopleContainer);
    }
    ```

1. Below the `QueryMember` method create a new method that looks like this:

    ```csharp
    private static async Task<double> ReadMember(Container peopleContainer)
    {

    }
    ```

1. Add the following code to use the `ReadItemAsync` method of the `Container` class to retrieve an item using the unique id and the partition key set to the last name from the previous step. Replace the `example.document` and `<Last Name>` tokens:

    ```csharp
    ItemResponse<object> response = await peopleContainer.ReadItemAsync<object>("example.document", new PartitionKey("<Last Name>"));
    ```

1. Add the following line of code to print out the value of the **RequestCharge** property of the `ItemResponse<T>` instance and return the Request Charge (we will use this value in a later exercise):

    ```csharp
    await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
    return response.RequestCharge;
    ```

1. The method should now look similar to this:

    ```csharp
    private static async Task<double> ReadMember(Container peopleContainer)
    {
        ItemResponse<object> response = await peopleContainer.ReadItemAsync<object>("372a9e8e-da22-4f7a-aff8-3a86f31b2934", new PartitionKey("Batz"));
        await Console.Out.WriteLineAsync($"{response.RequestCharge} RU/s");
        return response.RequestCharge;
    }
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > You should see that the direct read consumed significantly fewer RUs—**1 RU** compared to **~2.82 RUs** for the query. This ~2.8x efficiency gain demonstrates why **ReadItemAsync()** is the preferred method when you have both the item's `id` and partition key value:
    >
    > - **Query approach** (`SELECT TOP 1 * FROM c WHERE c.id = '...'`): **2.82 RUs** — The query engine must parse the SQL, create an execution plan, and search the partition even though the id is unique.
    > - **Direct read approach** (`ReadItemAsync(id, partitionKey)`): **1 RU** — Bypasses the query engine entirely and goes directly to the backend store to retrieve the item by its unique coordinates.
    >
    > A direct read of 1 KB of data or less will always cost exactly **1 RU**, making it the most efficient way to retrieve a single item when you have its id and partition key.

## Setting Throughput for Expected Workloads

Using appropriate RU/s settings for container or database throughput can allow you to meet desired performance at minimal cost. Deciding on a good baseline and varying settings based on expected usage patterns are both strategies that can help.

### Estimating Throughput Needs

1. In the **Azure Cosmos DB** blade, locate and select the **Metrics** link on the left side of the blade under the **Monitoring** section.
1. Observe the values in the **Number of requests** graph to see the volume of requests your lab work has been making to your Cosmos containers.

    ![The Metrics dashboard is displayed](../media/09-metrics.jpg "Review your Cosmos DB metrics dashboard")

    > Various parameters can be changed to adjust the data shown in the graphs and there is also an option to export data to csv for further analysis. For an existing application this can be helpful in determining your query volume.

1. Return to the Visual Studio Code window and locate the `Main` method. Add a new line `await EstimateThroughput(peopleContainer);` to look like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        //await CreateTransactions(transactionContainer);
        //await QueryTransactions(transactionContainer);
        //await QueryMember(peopleContainer);
        //await ReadMember(peopleContainer);
        await EstimateThroughput(peopleContainer);

    }
    ```

1. At the bottom of the class add a new method `EstimateThroughput` with the following code:

    ```csharp
    private static async Task EstimateThroughput(Container peopleContainer)
    {

    }
    ```

1. Add the following lines of code to this method. These variables represent the estimated workload for our application:

    ```csharp
    int expectedWritesPerSec = 200;
    int expectedReadsPerSec = 800;
    ```

    > These types of numbers could come from planning a new application or tracking actual usage of an existing one. Details of determining workload are outside the scope of this lab.

1. Next add the following lines of code call `CreateMember` and `ReadMember` and capture the Request Charge returned from them. These variables represent the actual cost of these operations for our application:

    ```csharp
    double writeCost = await CreateMember(peopleContainer);
    double readCost = await ReadMember(peopleContainer);
    ```


1. Add the following line of code as the last line of this method to print out the estimated throughput needs of our application based on our test queries:

    ```csharp
    await Console.Out.WriteLineAsync($"Estimated load: {writeCost * expectedWritesPerSec + readCost * expectedReadsPerSec} RU/s");
    ```

1. The `EstimateThroughput` method should now look like this:

    ```csharp
    private static async Task EstimateThroughput(Container peopleContainer)
    {
        int expectedWritesPerSec = 200;
        int expectedReadsPerSec = 800;

        double writeCost = await CreateMember(peopleContainer);
        double readCost = await ReadMember(peopleContainer);

        await Console.Out.WriteLineAsync($"Estimated load: {writeCost * expectedWritesPerSec + readCost * expectedReadsPerSec} RU/s");
    }
    ```

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > Based on your test results, the application will require approximately **10,780 RU/s** of throughput capacity:
    >
    > - **Expected writes per second**: 200
    > - **Expected reads per second**: 800
    > - **Write cost**: 49.9 RU (creating a member document)
    > - **Read cost**: 1 RU (direct read with id and partition key)
    > - **Estimated load**: (49.9 × 200) + (1 × 800) = **10,780 RU/s**
    >
    > This calculation demonstrates how to size your Cosmos DB throughput for expected workloads. To get the most accurate estimate for RU/s needs for your applications, follow the same pattern: measure the RU cost for each operation type, multiply by the expected frequency per second, and sum across all operations. Alternatively, you can use the **Metrics** tab in the Azure portal to measure actual average throughput for existing applications.

### Adjusting for Usage Patterns

Many applications have workloads that vary over time in a predictable way. Programmatically adjusting throughput allows you to optimize costs while maintaining performance.

**When to use programmatic throughput adjustment:**

1. **Time-based workload patterns**:
   - Business applications with heavy workload during business hours (9-5) but minimal usage overnight
   - E-commerce sites with peak traffic during sales events or holiday seasons
   - Reporting systems that process batches at specific times (end of day, monthly)

2. **Event-driven scaling**:
   - Data migration operations requiring temporary throughput increases
   - Batch processing jobs that need higher throughput during execution
   - Scheduled maintenance windows requiring reduced throughput

3. **Cost optimization strategies**:
   - Scale down during known low-traffic periods to reduce costs
   - Scale up proactively before anticipated load increases
   - Respond to monitoring alerts when approaching RU limits

4. **When NOT to use programmatic adjustment**:
   - For unpredictable, rapidly changing workloads → Use **autoscale** instead
   - For steady-state workloads → Use fixed **manual** throughput
   - When you need instant scaling → Autoscale responds faster than programmatic changes

**Important**: Throughput changes take time to propagate (typically seconds to minutes depending on scale). Plan adjustments ahead of anticipated load changes rather than reacting in real-time. For automatic, immediate scaling, configure **autoscale throughput** instead of manual programmatic adjustments.

1. Locate the `Main` method and comment out the last line and add a new line `await UpdateThroughput(peopleContainer);` so it looks like this:

    ```csharp
    public static async Task Main(string[] args)
    {

        Database database = _client.GetDatabase(_databaseId);
        Container peopleContainer = database.GetContainer(_peopleContainerId);
        Container transactionContainer = database.GetContainer(_transactionContainerId);

        //await CreateMember(peopleContainer);
        //await CreateTransactions(transactionContainer);
        //await QueryTransactions(transactionContainer);
        //await QueryMember(peopleContainer);
        //await ReadMember(peopleContainer);
        //await EstimateThroughput(peopleContainer);
        await UpdateThroughput(peopleContainer);
    }
    ```

1. At the bottom of the class create a new method `UpdateThroughput`:

    ```csharp
    private static async Task UpdateThroughput(Container peopleContainer)
    {

    }
    ```

1. Add the following code to retrieve the current RU/sec setting for the container:

    ```csharp
    int? throughput = await peopleContainer.ReadThroughputAsync();
    await Console.Out.WriteLineAsync($"{throughput} RU/s");
    ```

    > Note that the type of the **Throughput** property is a nullable value. Provisioned throughput can be set either at the container or database level. If set at the database level, this property read from the **Container** will return null. When set at the container level, the same method on **Database** will return null.

1. Add the following line of code to print out the minimum throughput value for the container:

    ```csharp
    ThroughputResponse throughputResponse = await container.ReadThroughputAsync(new RequestOptions());
    int? minThroughput = throughputResponse.MinThroughput;
    await Console.Out.WriteLineAsync($"Minimum Throughput {minThroughput} RU/s");
    ```

    > Although the overall minimum throughput that can be set is 400 RU/s, specific containers or databases may have higher limits depending on size of stored data, previous maximum throughput settings, or number of containers in a database. Trying to set a value below the available minimum will cause an exception here. The current allowed minimum value can be found on the **ThroughputResponse.MinThroughput** property.
    
    > **Important**: Your `PeopleCollection` may show a minimum throughput of **1000 RU/s or higher** instead of 400 RU/s. This is normal and demonstrates a key concept: minimum throughput increases based on your container's data size and usage history. Always check the `MinThroughput` property before scaling down.

1. Add the following code to update the RU/s setting for the container then print out the updated RU/s for the container. **Note**: Adjust the throughput value to match or exceed your container's minimum (use the value from `MinThroughput` above):

    ```csharp
    // Use your actual minimum throughput value here (e.g., 1000 if that's your minimum)
    await peopleContainer.ReplaceThroughputAsync(1000);
    throughput = await peopleContainer.ReadThroughputAsync();
    await Console.Out.WriteLineAsync($"New Throughput {throughput} RU/s");
    ```

1. The finished method should look like this:

    ```csharp
    private static async Task UpdateThroughput(Container peopleContainer)
    {
        ThroughputResponse response = await peopleContainer.ReadThroughputAsync(new RequestOptions());
        
        // Check if autoscale is enabled
        if (response.Resource.AutoscaleMaxThroughput.HasValue)
        {
            await Console.Out.WriteLineAsync($"Current: Autoscale with max {response.Resource.AutoscaleMaxThroughput} RU/s");
            await Console.Out.WriteLineAsync($"Minimum allowed: {response.MinThroughput} RU per sec");
            
            // To change autoscale throughput, use autoscale settings
            ThroughputProperties autoscaleProperties = ThroughputProperties.CreateAutoscaleThroughput(4000);
            ThroughputResponse newResponse = await peopleContainer.ReplaceThroughputAsync(autoscaleProperties);
            await Console.Out.WriteLineAsync($"New Throughput: Autoscale with max {newResponse.Resource.AutoscaleMaxThroughput} RU/s");
        }
        else
        {
            // Manual throughput
            int? current = response.Resource.Throughput;
            await Console.Out.WriteLineAsync($"{current} RU per sec");
            await Console.Out.WriteLineAsync($"Minimum allowed: {response.MinThroughput} RU per sec");
            
            // Update to a new manual throughput value
            await peopleContainer.ReplaceThroughputAsync(1000);
            ThroughputResponse newResponse = await peopleContainer.ReadThroughputAsync(new RequestOptions());
            int? newThroughput = newResponse.Resource.Throughput;
            await Console.Out.WriteLineAsync($"New Throughput {newThroughput} RU/s");
        }
    }
    ```

    > This method handles both **autoscale** and **manual** throughput modes. Autoscale containers require using `ThroughputProperties.CreateAutoscaleThroughput()` to update the maximum throughput, while manual containers use `ReplaceThroughputAsync()` with a fixed RU/s value.

1. Save all of your open editor tabs.

1. In the terminal pane, enter and execute the following command:

    ```sh
    dotnet run
    ```

1. Observe the output of the console application.

    > Your output will depend on whether your container uses **autoscale** or **manual** throughput:
    >
    > **Autoscale example:**
    > ```
    > Current: Autoscale with max 1000 RU/s
    > Minimum allowed: 1000 RU per sec
    > New Throughput: Autoscale with max 4000 RU/s
    > ```
    >
    > **Manual throughput example:**
    > ```
    > 400 RU per sec
    > Minimum allowed: 400 RU per sec
    > New Throughput 1000 RU/s
    > ```
    >
    > The output shows the initial provisioned throughput before and after the update. Autoscale containers display the maximum RU/s, while manual containers show the fixed RU/s allocation.

1. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.

1. In the **Data Explorer** section, expand the **FinancialDatabase** database node, expand the **PeopleCollection** node, and then select the **Scale & Settings** option.

1. In the **Settings** section, locate the **Throughput** field and verify the updated value:
    - **Autoscale**: Maximum RU/s is now set to **4000**
    - **Manual**: Fixed RU/s is now set to **1000**

> Note that you may need to refresh the Data Explorer to see the new value.

> If this is your final lab, follow the steps in [Removing Lab Assets](11-cleaning_up.md) to remove all lab resources.
