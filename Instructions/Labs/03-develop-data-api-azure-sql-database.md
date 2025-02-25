---
lab:
    title: 'Develop a Data API for Azure SQL Database'
    module: 'Develop a Data API for Azure SQL Database'
---

# Develop a Data API for Azure SQL Database

In this exercise, you will develop and deploy a data API for an Azure SQL Database using Azure Static Web Apps. This will provide hands-on experience in setting up a Data API Builder configuration and deploying it within an Azure Static Web App environment.

## Prerequisites

Before starting this exercise, ensure you have the following:

- An active Azure subscription.
- Basic knowledge of Azure SQL Database, Azure Static Web Apps, and GitHub.
- Visual Studio Code installed with the necessary extensions.
- A GitHub account for managing the repository.

## Set up the environment

There are a few steps you need to take to set up the environment for this exercise.

### Install the Visual Studio Code extensions

Before you can start this exercise, you need to install the Visual Studio Code extensions.

1. Open Visual Studio Code.
1. Open up a terminal window in Visual Studio Code.
1. Install the Static Web Apps CLI by running the following command:

    ```bash
    npm install -g @azure/static-web-apps-cli
    ```

1. Install the Data API Builder CLI by running the following command:

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

Visual Studio Code is now set up with the necessary extensions.

### Create an Azure SQL Database

If you haven't already done so, you need to create an Azure SQL Database.

1. Sign in to the [Azure portal](https://portal.azure.com?azure-portal=true). 
1. Navigate to the **Azure SQL** page, and then select **+ Create**.
1. Select **SQL Database**, *Single database* and the **Create** button.
1. Fill in the required information on the **Create SQL Database** dialog and select **OK** (leave all other options to their default).

    | Setting | Value |
    | --- | --- |
    | Free serverless offer | *Apply offer* |
    | Subscription | Your subscription |
    | Resource group | *Select or create a new resource group* |
    | Database name | *MyDB* |
    | Server | *Select the **Create new** link* |
    | Server name | *Choose a unique name* |
    | Location | *Select a location* |
    | Authentication method | *Use SQL authentication* |
    | Server admin login | *sqladmin* |
    | Password | *Enter a password* |
    | Confirm password | *Confirm the password* |

1. Select **Review + create**, and then **Create**.
1. After the deployment is complete, navigate to the Azure SQL Database *server* you created.
1. Select **Networking** under **Security** on the left pane. Add your IP address to the firewall rules.
1. Select **Save**.

### Add Sample Data to the Database

Now that you have an Azure SQL Database, you need to add some sample data. This will help you test the API once it's up and running.

1. Navigate to your newly created Azure SQL Database.
1. Use the **Query editor** in the Azure portal to run the following SQL script:

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    
    INSERT INTO [dbo].[Employees] VALUES (1,'John', 'Smith', 'Accounting');
    INSERT INTO [dbo].[Employees] VALUES (2,'Michelle', 'Sanchez', 'IT');
    
    SELECT * FROM [dbo].[Employees];
    ```

### Create a basic web app in GitHub

Before we can create an Azure Static Web App, we need to create a basic web app in GitHub.

1. To create a basic web app in GitHub, go to the [generate a vanilla website](https://github.com/staticwebdev/vanilla-basic/generate).
1. Make sure the Repostiory template is set to **staticwebdev/vanilla-basic**.
1. Under ***Owner***, select your account.
1. Under ***Repository name***, enter the name **my-sql-repo**.
1. Make the repository **Private**.
1. Select the **Create repository** button.

## Create an Azure Static Web App

We will first create our static web app and then add the Data API Builder configuration to it.

1. On the Azure portal, navigate to the **Static Web Apps** page.
1. Select **+ Create**.
1. Fill in the following information on the **Create Static Web App** dialog (leave all other options to their default):

    | Setting | Value |
    | --- | --- |
    | Subscription | Your subscription |
    | Resource group | *Select or create a new resource group* |
    | Name | *a unique name* |
    | Hosting plan Source | *GitHub* |
    | GitHub Account | *Select your account* |
    | Organization | *Most likely your GitHub user name* |
    | Repository | *Select the repository you created in the previous step* |
    | Branch | *main* |

1. Select **Review + create** and then **Create**.
1. Once it's deployed, go to the resource.
1. Select the **View app in browser** button, you should see a simple web page with a welcome message. You can close this tab.

## Add Data API Builder configuration file

It's time to add the Data API Builder configuration to the Azure Static Web App. We need to create a new file in the GitHub repository to add the Data API Builder configuration.

1. In Visual Studio Code, clone the GitHub repository you created earlier.
1. Open a terminal window in Visual Studio Code.
1. Run the following command to create a new Data API Builder configuration file:

    ```bash
    swa db init --database-type "mssql"
    ```

    This will create a new folder named *swa-db-connections* and a file named *staticwebapp.database.config.json* inside that folder.

1. Run the following command to add the database entities to the configuration file:

    ```bash
    dab add "Employees" --source "dbo.Employees" --permissions "anonymous:*" --config "swa-db-connections/staticwebapp.database.config.json"
    ```

1. Review the contents of the *staticwebapp.database.config.json* file. 
1. Commit and push the changes to the GitHub repository.

## Configure the database connection

1. On the Azure portal, navigate to the Azure Static Web App you created.
1. Under **Settings**, select **Database connection**.
1. Select **Link existing database**.
1. In the **Link database* dialog, select the Azure SQL Database you created earlier with the following additional settings.

    | Setting | Value |
    | --- | --- |
    | Database Type | *Azure SQL Database* |
    | Authentication Type | *Connection String* |
    | Username | *your admin user name* |
    | Password | *the password you gave you admin user* |
    | Acknowledgement checkbox | *Checked* |

   > **Note:** In a production environment, restrict access to only the necessary IP addresses. Additionally, consider using Managed Identities for your Static Web App to access the database instead of SQL authentication. For more information, see the [Managed identities in Microsoft Entra for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).
1. Select **Link**.

## Test the Data API endpoint

Now we only need to test the Data API endpoint.

1. On the Azure portal, navigate to the Azure Static Web App you created.
1. On the Overview page, copy the URL of the web app.
1. Open a new browser tab and paste the URL. You should still see the simple web page with the message **Vanilla JavaScript App**.
1. Add **/data-api** to the end of the URL and press **Enter**. It should display **Healthy** to indicate that the Data API is working.
1. Add **/data-api/rest/Employees** to the end of the URL and press **Enter**. You should see the sample data you added to the Azure SQL Database earlier.

You have successfully developed and deployed a data API for an Azure SQL Database using Azure Static Web Apps.

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in extra costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about Data API builder for Azure SQL Databases, see [What is Data API builder for Azure Databases?](https://learn.microsoft.com/azure/data-api-builder/overview?azure-portal=true).
