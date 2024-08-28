+++
title = "Azure Cosmos DB"
+++

## Azure Cosmos DB

Azure Cosmos DB is a fully managed NoSQL database for high-performance applications of any size or scale.  See [Azure Cosmos DB Provider](https://learn.microsoft.com/ef/core/providers/cosmos/) for information on using Azure Cosmos DB with Entity Framework Core.  When using Azure Cosmos DB with the Datasync Community Toolkit:

1. Set up the Cosmos Container with a composite index that specifies the `UpdatedAt` and `Id` fields.  Composite indices can be added to a container through the Azure portal, ARM, Bicep, Terraform, or within code. Here's an example [bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) resource definition:

    ``` bicep
    resource cosmosContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-04-15' = {
        name: 'TodoItems'
        parent: cosmosDatabase
        properties: {
            resource: {
                id: 'TodoItems'
                partitionKey: {
                    paths: [
                        '/Id'
                    ]
                    kind: 'Hash'
                }
                indexingPolicy: {
                    indexingMode: 'consistent'
                    automatic: true
                    includedPaths: [
                        {
                            path: '/*'
                        }
                    ]
                    excludedPaths: [
                        {
                            path: '/"_etag"/?'
                        }
                    ]
                    compositeIndexes: [
                        [
                            {
                                path: '/UpdatedAt'
                                order: 'ascending'
                            }
                            {
                                path: '/Id'
                                order: 'ascending'
                            }
                        ]
                    ]
                }
            }
        }
    }
    ```

   You can also review our [bicep module](https://github.com/CommunityToolkit/Datasync/blob/main/tests/infra/databases/cosmos.bicep) that we use for testing.

   If you pull a subset of items in the table, ensure you specify all properties involved in the query.

2. Derive models from the `CosmosEntityTableData` class:

    ``` csharp
    public class TodoItem : CosmosEntityTableData
    {
        public string Title { get; set; }
        public bool Completed { get; set; }
    }
    ```

3. Add an `OnModelCreating(ModelBuilder)` method to the `DbContext`.  The Cosmos DB driver for Entity Framework places all entities into the same container by default.  At a minimum, you must pick a suitable partition key and ensure the `EntityTag` property is marked as the concurrency tag.  For example, the following snippet stores the `TodoItem` entities in their own container with the appropriate settings for Azure Mobile Apps:

    ``` csharp
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.Entity<TodoItem>(builder =>
        {
            // Store this model in a specific container.
            builder.ToContainer("TodoItems");
            // Do not include a discriminator for the model in the partition key.
            builder.HasNoDiscriminator();
            // Set the partition key to the Id of the record.
            builder.HasPartitionKey(model => model.Id);
            // Set the concurrency tag to the EntityTag property.
            builder.Property(model => model.EntityTag).IsETagConcurrency();
        });
        base.OnModelCreating(builder);
    }
    ```

Azure Cosmos DB is supported in the `Microsoft.AspNetCore.Datasync.EFCore` NuGet package since v5.0.11. For more information, review the following links:

* [EF Core Azure Cosmos DB Provider](https://learn.microsoft.com/ef/core/providers/cosmos) documentation.
* [Cosmos DB index policy](https://learn.microsoft.com/azure/cosmos-db/index-policy) documentation.
* [Test Azure Cosmos DB Context](https://github.com/CommunityToolkit/Datasync/blob/main/tests/CommunityToolkit.Datasync.TestCommon/Databases/CosmosDb/CosmosDbContext.cs)