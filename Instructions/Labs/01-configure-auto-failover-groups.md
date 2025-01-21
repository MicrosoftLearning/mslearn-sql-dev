---
lab:
    title: 'Enable application resilience with auto-failover groups for Azure SQL Database'
    module: 'Get started with Azure SQL Database for cloud-native application development'
---

# Enable application resilience with auto-failover groups for Azure SQL Database

In this exercise you’ll create two Azure SQL databases that will act as primary and secondary. You'll configure auto-failover groups to ensure high availability and disaster recovery of your application databases, and validate the replication status on your application.

This exercise should take approximately **30** minutes to complete.

## Before you start

Before you can start this exercise, you'll need:

- An Azure subscription with appropriate permissions to create and manage resources.
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true) installed on your computer with the following extension installed:
    - [C# Dev Kit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?azure-portal=true).

## Create primary and secondary Azure SQL servers

First, we'll set up both primary and secondary servers, and we'll use the **AdventureWorksLT** sample database.

1. Sign in to the [Azure portal](https://portal.azure.com?azure-portal=true).

1. Select the Cloud Shell icon in the top-right corner of the Azure portal. It looks like a `>_` symbol. If prompted, choose **Bash** as the shell type.

1. Run the following commands in the Cloud Shell terminal. Replace the values `<your_resource_group>`, `<your_primary_server>`, `<your_location>`, `<your_secondary_server>`, and `<your_admin_password>` with your actual values:

    * Create a resource group
    ```azurecli
    az group create --name <your_resource_group> --location <your_location>
    ```

    * Create the primary SQL server
    ```azurecli        
    az sql server create --name <your_primary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>
    ```

    * Create the secondary SQL server. Same script, only change the server name and location
    ```azurecli
    az sql server create --name <your_secondary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>    
    ```

    * Create a sample database on the primary server with the specified pricing tier
    ```azurecli
    az sql db create --resource-group <your_resource_group> --server <your_primary_server> --name AdventureWorksLT --sample-name AdventureWorksLT --service-objective "S0"    
    ```
    
1. After the deployments are complete, navigate to the primary Azure SQL server you created.
1. Select **Networking** under **Security** on the left pane. Add your IP address to the firewall rules.
1. Select the **Allow Azure services and resources to access this server** option.
1. Select **Save**.
1. Repeat the steps above for the secondary server.

    These steps ensure that you have a structured and redundant Azure SQL Database environment ready for use.

## Configure auto-failover groups

Next, you’ll create an auto-failover group for the Azure SQL Database you previously set up. This involves establishing a failover group between two servers, and verifying the setup to ensure it's working properly.

1. Run the following commands on the Cloud Shell terminal. Replace the values `<your_failover_group>`, `<your_resource_group>`, `<your_primary_server>`, and `<your_secondary_server>` with your actual values:

    * Create the failover group
    ```azurecli
    az sql failover-group create -n <your_failover_group> -g <your_resource_group> -s <your_primary_server> --partner-server <your_secondary_server> --failover-policy Automatic --grace-period 1 --add-db AdventureWorksLT
    ```

    * Verify the failover group
    ```azurecli    
    az sql failover-group show -n <your_failover_group> -g <your_resource_group> -s <your_primary_server>
    ```

    > Take a moment to review the results and the `partnerServers` values. Why is this important?

    > By checking the `role` attribute within each partner server, you can determine whether a server is currently acting as the primary or secondary. This information is crucial for understanding the current configuration and readiness of the failover group. It helps you assess the potential impact on your application during failover scenarios and ensures that your setup is correctly configured for high availability and disaster recovery.
    
## Integrate with the application code

To connect your .NET application to the Azure SQL Database endpoint, you’ll need to follow these steps.

1. In Visual Studio Code, open the terminal and run the following commands to install the `Microsoft.Data.SqlClient` package and create a new .NET console application.

    ```bash
    dotnet new console -n AdventureWorksLTApp
    cd AdventureWorksLTApp 
    dotnet add package Microsoft.Data.SqlClient --version 5.2.1
    ```

1. Open the `AdventureWorksLTApp` folder that was created with the previous step in **Visual Studio Code**.

1. Create an `appsettings.json` file in the root of your project directory. This configuration file will store your database connection string. Be sure to replace the `<your_failover_group>` and `<your_password>` values in the connection string with your actual details.

    ```json
    {
      "ConnectionStrings": {
        "FailoverGroupConnection": "Server=tcp:<your_failover_group>.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      }
    }
    ```

1. Open the `.csproj` file in **Visual Studio Code** and add the following content right below the `</PropertyGroup>` tag.

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
    </ItemGroup>
    ```

    Your complete `.csproj` file should look similar to this.

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>
    
      <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
      </ItemGroup>
    
    </Project>
    ```

1. Open the `Program.cs` file in **Visual Studio Code**. In the editor, replace all the existing code with the code provided below.

    > **Note:** Take a moment to review the code and observe how it prints information about the primary and secondary servers in the auto-failover group.

    ```csharp
    using System;
    using Microsoft.Data.SqlClient;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    
    namespace AdventureWorksLTApp
    {
        class Program
        {
            static void Main(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();
    
                string connectionString = configuration.GetConnectionString("FailoverGroupConnection");
    
                ExecuteQuery(connectionString);
            }
    
            static void ExecuteQuery(string connectionString)
            {
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    try
                    {
                        connection.Open();
                        string query = @"
                            SELECT 
                                @@SERVERNAME AS [Primary_Server],
                                partner_server AS [Secondary_Server],
                                partner_database AS [Database],
                                replication_state_desc
                            FROM 
                                sys.dm_geo_replication_link_status";
    
                        using (SqlCommand command = new SqlCommand(query, connection))
                        {
                            using (SqlDataReader reader = command.ExecuteReader())
                            {
                                while (reader.Read())
                                {
                                    Console.WriteLine($"Primary Server: {reader["Primary_Server"]}");
                                    Console.WriteLine($"Secondary Server: {reader["Secondary_Server"]}");
                                    Console.WriteLine($"Database: {reader["Database"]}");
                                    Console.WriteLine($"Replication State: {reader["replication_state_desc"]}");
                                }
                            }
                        }
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error executing query: {ex.Message}");
                    }
                    finally
                    {
                        connection.Close();
                    }
                }
            }
        }
    }
    ```

1. Run the code by selecting **Run** > **Start Debugging** from the menu, or simply press **F5**. You can also select the play button in the top toolbar to start the application.

    > **Important:** If you receive a message *"You don't have an extension for debugging C#. Should we find a C# extension in the Marketplace?"*, ensure that the **C# Dev Kit** extension is installed.

1. After running the code, you should see the output in the **Debug Console** tab in Visual Studio Code.

    ```
    Primary Server: <your_server_name>
    Secondary Server: <your_server_name>
    Database: AdventureWorksLT
    Replication State: CATCH_UP
    ```
    
    The replication state `CATCH_UP` means that the database is fully synchronized with its partner and is ready for failover. Monitoring the replication state can help in identifying performance bottlenecks and ensuring that data replication is happening efficiently.

## Fail over to a secondary region

Imagine a scenario where the primary Azure SQL Database is experiencing issues due to a regional outage. To maintain service continuity and minimize downtime, you need to switch your application to the secondary replica by performing a forced failover.

During a forced failover, all new TDS sessions are automatically re-routed to the secondary server, which then becomes the primary server. The best part is that you don’t need to change the application connection string, as the endpoint remains the same.

Let’s initiate a failover and run our application to check the status of our primary and secondary servers.

1. Go back to Azure portal, open a new instance of Cloud Shell terminal. Run the following code. Replace the values `<your_failover_group>`, `<your_resource_group>`, and `<your_primary_server>` with your actual values. The `--server` parameter value should be the current secondary.

    ```azurecli
    az sql failover-group set-primary --name <your_failover_group> --resource-group <your_resource_group> --server <your_server_name>
    ```

    > **Note**:  This operation might take a few minutes.

1. Once the failover is complete, run the application again to verify the replication status. You should see that the secondary server has now taken over as the primary, and the original primary server has become the secondary.

Consider why you might want to place your primary and secondary application databases in the same region, and when it could be beneficial to choose different regions.

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in extra costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about auto-failover groups for Azure SQL Database, see [Failover groups overview & best practices (Azure SQL Database)](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?azure-portal=true).
