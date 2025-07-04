---
lab:  
    title: 'Import and export data for development in Azure SQL Database'  
    module: 'Import and export data for development in Azure SQL Database'  
---

# Import and export data for development in Azure SQL Database

In this exercise, you will import data from an external REST endpoint (simulated using Azure Static Web App) and export data using an Azure Function. The lab will provide practical experience in working with Azure SQL Database for development purposes, focusing on integrating REST APIs and Azure Functions to handle data import/export operations.

## Prerequisites

Before starting this lab, ensure you have the following:

- An active Azure subscription with permission to create and manage resources.
- Basic knowledge of Azure SQL Database, REST APIs, and Azure Functions.
- Visual Studio Code installed with the following extensions:
      - Azure Functions extension.
- Git installed for cloning the repository.
- SQL Server Management Studio (SSMS) or Azure Data Studio for managing the database.

## Set up the environment

Let’s begin by setting up the necessary resources for this lab, including an Azure SQL Database and the tools needed to import and export data.

### Create an Azure SQL Database

This step requires you to create a database in Azure:

1. Sign in to the [Azure portal](https://portal.azure.com?azure-portal=true).
1. In the Azure portal, go to the **SQL Databases** page.
1. Select **Create**.
1. Fill out the required fields:

    | Setting | Value |
    |---|---|
    | Free serverless offer | Apply offer |
    | Subscription | Your subscription |
    | Resource group | *Select or create a new resource group* |
    | Database name | **MyDB** |
    | Server | *Select the **Create new** link* |
    | Server name | *Choose a unique name* |
    | Location | *Select a location* |
    | Authentication method | SQL authentication |
    | Server admin login | **sqladmin** |
    | Password | *Enter a secure password* |
    | Confirm password | *Confirm the password* |

1. Select **Review + Create**, then **Create**.
1. After the deployment finishes, navigate to the **Networking** section of your ***Azure SQL Server*** (not the Azure SQL Database) and:
    1. Add your IP address to the firewall rules. This will allow you to use SQL Server Management Studio (SSMS) or Azure Data Studio for managing the database.
    1. Select the **Allow Azure services and resources to access this server** checkbox. This will allow the Azure Function App to access the database server.
    1. Save your changes.

> [!NOTE]
> In a production environment you will need to determine which type of access and from where you want to grant access to. While the function will have a slight change if you choose Entra authentication only, note that you will still need to enable the *Allow Azure services and resources to access this server* to allow the Azure Function App to access the server.

### Clone the GitHub repository

1. Open **Visual Studio Code**.

1. Clone the GitHub Repo and prepare your project:

    1. In **Visual Studio Code**, open the **Command Palette** by pressing **Ctrl+Shift+P** (Windows) or **Cmd+Shift+P** (Mac).
    1. Type **Git: Clone** and select **Git: Clone**.
    1. In the prompt, enter the following URL to clone the repository:
        ```bash
        https://github.com/MicrosoftLearning/mslearn-sql-dev.git
        ```

    1. Choose the destination folder where you'd like to clone the repository.

### Set Up Azure Blob Storage for JSON data

We’ll now set up **Azure Blob Storage** to host the **employees.json** file. Follow these steps within the Azure portal and **Visual Studio Code**.

Let's start by creating an Azure Storage Account.

1. In the **Azure portal**, go to the **Storage Accounts** page.
1. Select **Create**.
1. Fill out the required fields:

    | Setting | Value |
    |---|---|
    | Subscription | Your subscription |
    | Resource group | Select or create a new resource group |
    | Storage account name | Choose a globally unique name |
    | Region | Choose the region closest to you |
    | Primary service | **Azure Blob Storage or Azure Data Lake Storage Gen2** |
    | Performance | Standard |
    | Redundancy | Locally-redundant storage (LRS) |

1. Select **Review + Create**, then **Create**.
1. Wait for the storage account to be created.

Now that we have an account, let's upload **employees.json** to Blob Storage.

1. Go to the **Storage Accounts** page in the Azure portal.
1. Select your storage account.
1. Navigate to the **Containers** section.
1. Create a new container named **jsonfiles**.
1. Inside the container, click **Upload** and upload the **employees.json** file located under the **/Allfiles/Labs/04/blob-storage** in the cloned directory.

While we could allow anonymous access to the file, in our case, let's generate a *Shared Access Signature (SAS)* for this file to ensure secure access.

1. On the **jsonfiles** container, select the **employees.json** file.
1. Select **Generate SAS** from the file's context menu.
1. Review the settings and select **Generate SAS and URL**.
1. A Blob SAS token and Blob SAS URL will be generated. Copy the **Blob SAS token** and the **Blob SAS URL** for use in the next steps. You won't be able to access the token value again once you close that window.

We should now have a secure URL to access the **employees.json** file, let's go ahead and test it.

1. Open a new browser tab and paste the **Blob SAS URL**.
1. You should see the contents of the **employees.json** file displayed in the browser, which should look like this:

    ```json
    {
        "employees": [
            {
                "EmployeeID": 1,
                "FirstName": "John",
                "LastName": "Doe",
                "Department": "HR"
            },
            {
                "EmployeeID": 2,
                "FirstName": "Jane",
                "LastName": "Smith",
                "Department": "Engineering"
            }
        ]
    }
    ```

### Import data from the blob storage to Azure SQL Database

We are now ready to import the data from the **employees.json** file hosted on Azure Blob Storage to our Azure SQL Database.

We need to start by creating a **Master Key** and a **Database Scoped Credential** in the Azure SQL Database.

1. Connect to your Azure SQL Database using **SQL Server Management Studio** (SSMS) or **Azure Data Studio**.
1. Run the following SQL command to create a master key *if you don't already have one*:

    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword!';
    ```

1. Next, create a **Database Scoped Credential** to access the Azure Blob Storage by running the following SQL command:

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL MyBlobCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', 
    SECRET = '<your-sas-token>';
    ```

    Replace \<your-sas-token\> with the Blob SAS token generated earlier.

1. Finally you need a **Data Source** to access the Azure Blob Storage. Run the following SQL command to create a **Data Source**:

    ```sql
    CREATE EXTERNAL DATA SOURCE MyBlobDataSource
    WITH (
        TYPE = BLOB_STORAGE,
        LOCATION = 'https://<your-storage-account-name>.blob.core.windows.net',
        CREDENTIAL = MyBlobCredential
    );
    ```

    Replace \<your-storage-account-name\> with the name of your Azure Storage Account.

Everything is now set up to import the data from the **employees.json** file into the *Azure SQL Database*.

Use the following SQL command to import data from the **employees.json** file hosted on *Azure Blob Storage*:

```sql
SELECT EmployeeID
    , FirstName
    , LastName
    , Department
INTO dbo.employee_data
FROM OPENROWSET(
    BULK 'jsonfiles/employees.json',
    DATA_SOURCE = 'MyBlobDataSource',
    SINGLE_CLOB
) AS JSONData
CROSS APPLY OPENJSON(JSONData.BulkColumn, '$.employees')
WITH (
    EmployeeID INT '$.EmployeeID',
    FirstName NVARCHAR(50) '$.FirstName',
    LastName NVARCHAR(50) '$.LastName',
    Department NVARCHAR(50) '$.Department'
) AS EmployeeData;
```

This command reads the **employees.json** file from the **jsonfiles** container in the *Azure Blob Storage* and imports the data into the **employee_data** table in the Azure SQL Database.

You can now run the following SQL command to verify the data import: 

```sql
SELECT * FROM dbo.employee_data;
```

You should see the data from the **employees.json** file imported into the **employee_data** table.

---

## Export data using an Azure Function App

In this part of the lab, you’ll create an Azure Function App in C# to export data from your Azure SQL Database. This function will retrieve the data and return it as a JSON response.

### Create an Azure Function App in Visual Studio Code

Let's start by creating an Azure Function App in Visual Studio Code:

1. Open **Visual Studio Code**.
1. On the explorer pane, navigate to the **/Allfiles/Labs/04/azure-functions** folder.
1. Right-click the **azure-functions** folder and select **Open in Integrated Terminal**.
1. In the VS Code terminal, log in to Azure using the following command:

    ```bash
    az login
    ```

1. (Optional) If you have multiple subscriptions, set the active subscription:

    ```bash
    az account set --subscription <your-subscription-id>
    ```

1. Run the following command to create an Azure Function App:

    ```bash
    $functionappname = "YourUniqueFunctionAppName"
    $resourcegroup = "YourResourceGroupName"
    $location = "YourLocation"
    # NOTE - The following should be a new storage account name where your Azure function will reside.
    # It should not be the same Storage Account name used to store the JSON file
    $storageaccount = "YourStorageAccountName"

    az storage account create --name $storageaccount --location $location --resource-group $resourcegroup --sku Standard_LRS
    
    az functionapp create --resource-group $resourcegroup --consumption-plan-location $location --runtime dotnet --name  $functionappname --os-type Linux --storage-account $storageaccount --functions-version 4
    
    ```

    ***Replace the placeholders with your own values. Don't use the storage account name used for the json file, this script needs to create a new storage account to store the Azure Function App***.


### Create a new function app in Visual Studio Code

Let's create a new function in Visual Studio Code to export data from the Azure SQL Database:

You might need to add the Azure Functions extension to Visual Studio Code if you haven't already. You can do this by searching for **Azure Functions** in the extensions pane and installing it.

1. In Visual Studio Code, press **Ctrl+Shift+P** (Windows) or **Cmd+Shift+P** (Mac) to open the command palette.
1. Type and select **Azure Functions: Create New Project**.
1. Choose your **Function App** directory. Pick the **/Allfiles/Labs/04/azure-functions** folder of the GitHub clone repo.
1. Choose **C#** as the language.
1. Choose **.Net 8.0 LTS** as the runtime.
1. Choose **HTTP trigger** as the template.
1. Call the function **ExportDataFunction**.
1. Make the namespace **Contoso.ExportFunction**.
1. Give the function the **anonymous** access level.

### Write the C# code for exporting data

1. The Azure Function App might need a few packages to be installed first. You can install them by running the following commands:

    ```bash
    dotnet add package Microsoft.Data.SqlClient
    dotnet add package Newtonsoft.Json
    dotnet restore
    
    npm install -g azure-functions-core-tools@4 --unsafe-perm true

    ```

1. Replace the placeholder function code with the following C# code to query your Azure SQL Database and return the results as JSON:

    ```csharp
    using System;
    using System.IO;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Microsoft.Data.SqlClient;
    using Newtonsoft.Json;
    using System.Collections.Generic;
    using System.Threading.Tasks;
    
    public static class ExportDataFunction
    {
        [FunctionName("ExportDataFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
            ILogger log)
        {
            // Connection string to the database
            // NOTE: REPLACE THIS CONNECTION STRING WITH THE CONNECTION STRING OF YOUR AZURE SQL DATABASE
            string connectionString = "Server=tcp:yourserver.database.windows.net;Database=DataLabDatabase;User ID=youruserid;Password=yourpassword;Encrypt=True;";
            
            // List to hold employee data
            List<Employee> employees = new List<Employee>();
            
            try
            {
                // Establishing connection to the database
                using (SqlConnection conn = new SqlConnection(connectionString))
                {
                    await conn.OpenAsync();
                    var query = "SELECT EmployeeID, FirstName, LastName, Department FROM employee_data";
                    
                    // Executing the query
                    using (SqlCommand cmd = new SqlCommand(query, conn))
                    {
                        // Adding parameters to the query (if needed)
                        // cmd.Parameters.AddWithValue("@ParameterName", parameterValue);

                        using (SqlDataReader reader = await cmd.ExecuteReaderAsync())
                        {
                            // Reading data from the database
                            while (await reader.ReadAsync())
                            {
                                employees.Add(new Employee
                                {
                                    EmployeeID = (int)reader["EmployeeID"],
                                    FirstName = reader["FirstName"].ToString(),
                                    LastName = reader["LastName"].ToString(),
                                    Department = reader["Department"].ToString()
                                });
                            }
                        }
                    }
                }
            }
            catch (SqlException ex)
            {
                // Logging SQL errors
                log.LogError($"SQL Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
            catch (System.Exception ex)
            {
                // Logging unexpected errors
                log.LogError($"Unexpected Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
    
            // Returning the list of employees as a JSON response
            return new OkObjectResult(JsonConvert.SerializeObject(employees, Formatting.Indented));
        }
    
        // Employee class to hold employee data
        public class Employee
        {
            public int EmployeeID { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string Department { get; set; }
        }
    }
    ```

    *Remember to replace the **connectionString** with the connection string to your Azure SQL Database and input your sqladmin password in the connection string as well.*

    > **Note:** In a production environment, restrict access to only the necessary IP addresses. Additionally, consider using Managed Identities for your Azure Function App to access the database instead of SQL authentication. For more information, see the [Managed identities in Microsoft Entra for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).

1. Save the function code and ensure that your **.csproj** file includes the **Newtonsoft.Json** package for serializing objects to JSON. If it's not included, add it:

    ```xml
    <PackageReference Include="Newtonsoft.Json" Version="13.X.X" />
    ```

Time to deploy the Azure Function App to Azure.

### Deploy the Azure Function App to Azure

1. In the **Visual Studio Code** integrated terminal, run the following command to deploy the Azure Function to Azure:

    ```bash
    func azure functionapp publish <your-function-app-name>
    ```

    Replace ***<your-function-app-name>*** with the name of your Azure Function App.

1. Wait for the deployment to complete.

### Get the Azure Function App URL

1. Open the Azure portal and navigate to your Azure Function App.
1. In the *Overview* section, under the *Function* tab, you will see your new function listed, select it.
1. Under the **Code + Test** tab, select **Get function URL**.
1. Copy the **default (Function key)**, we will need it shortly. The URL should look something like this:
   
   ```url
   https://YourFunctionAppName.azurewebsites.net/api/ExportDataFunction?code=2pjO0HqRyz_13DHQg8ga-ysdDWbDU_eHdtlixbAHLVEGAzFuplomUg%3D%3D
   ```

### Test the Azure Function App

1. Once the deployment is complete, you can test the function by sending an HTTP request to the Function key URL you copied earlier from the Visual Studio Code terminal:

    ```bash
    curl https://<your-function-app-name>.azurewebsites.net/api/ExportDataFunction?code=<the function key embedded to your function URL>
    ```

1. The response should contain the exported data from your ***employee_data*** table in JSON format.

While this function is a simple example, you can extend it to include more complex logic and data processing, such as filtering, sorting, and aggregating data, and more.  Your code could also be extended to include error handling, logging, and security features.

### Clean up the resources

After completing the lab, you can delete the resources created in this exercise to avoid incurring additional costs:

- Delete the Azure SQL Database.
- Delete the Azure Storage Account.
- Delete the Azure Function App.
- Delete the resource group containing the resources.
