---
lab:
    title: 'Configure and Deploy CI/CD Pipelines for Azure SQL Database Projects'
    module: 'Develop for an Azure SQL Database'
---

# Configure and deploy CI/CD pipelines for Azure SQL Database projects

In this exercise you'll create, configure, and deploy CI/CD pipelines for Azure SQL Database projects using Visual Studio Code and GitHub Actions. This allows you to familiarize yourself with the process of setting up CI/CD pipelines for Azure SQL Database projects.

This exercise should take approximately **30** minutes to complete.

## Before you start

Before you can start this exercise, you need:

- An Azure subscription with appropriate permissions to create and manage resources.
- [Visual Studio Code](https://code.visualstudio.com/download) installed on your computer with the following extensions:
  - [SQL Database Projects](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql).
  - [GitHub Pull Requests](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github).
- GitHub accounts.
- Basic knowledge of *GitHub Actions* pipelines.

## Create an Azure SQL Database

First, you need to create a new Azure SQL Database.

1. Sign in to the [Azure portal](https://portal.azure.com?azure-portal=true). 
1. Navigate to the **Azure SQL** page, and then select **Create**.
1. Select **SQL Database**, *Single database* and the **Create** button.
1. Fill in the required information on the **Create SQL Database** dialog and select **OK**, leaving all other options at their default settings.

    | Setting | Value |
    | --- | --- |
    | Free serverless offer | *Apply offer* |
    | Subscription | Your subscription |
    | Resource group | *Select or create a new resource group* |
    | Database name | *MyDB* |
    | Server | *Create a new server* |
    | Server admin login | *Select a unique name* |
    | Location | *Select a location* |
    | Authentication method | *Use SQL authentication* |
    | Server admin login | *sqladmin* |
    | Password | *Enter a password* |
    | Confirm password | *Confirm the password* |

1. Select **Review + create**, and then **Create**.
1. After the deployment is complete, navigate to the Azure SQL Database *server* you created.
1. Select **Networking** under **Security** on the left pane. Add your IP address to the firewall rules.
1. Select the **Allow Azure services and resources to access this server** option. This option allows GitHub Actions to access the database.

    > **Note:** In a production environment, restrict access to only the necessary IP addresses. Additionally, consider using Managed Identities for your GitHub Action to access the database instead of SQL authentication. For more information, see the [Managed identities in Microsoft Entra for Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).

1. Select **Save**.

## Set up a GitHub repository

Next, you need to set up a new GitHub repository.

1. Open the [GitHub](https://github.com) website.
1. Sign in to your GitHub account.
1. Go to **Repositories** under your account, and select **New**.
1. Select your account for the **Owner**. Enter the name **my-sql-db-repo**.
1. Set the repository **Private**.
1. Select **Create repository**.

### Install the Visual Studio Code extensions and clone the repository

Before cloning the repository, ensure you have installed the necessary **Visual Studio Code** extensions. Refer to the **Before you start** section for guidance.

1. In Visual Studio Code, select **View** > **Command Palette**.
1. In the command palette, type `Git: Clone`, and select it.
1. Enter the URL of the repository you created in the previous step and select **Clone**. Your URL should follow this format: *https://github.com/<your_account>/<your_repository>.git*
1. Select or create a folder to store your repository files.

## Create and configure an Azure SQL Database project

An Azure SQL Database project in Visual Studio enables you to develop, build, test, and publish your database schema and data. In this section, you’ll create a project and configure it to connect to the Azure SQL Database you set up earlier.

1. In Visual Studio Code, select **View** > **Command Palette**.
1. In the command palette, type `Database projects: New` and select it.
    > **Note:** It may take a few minutes to install the SQL tools service for the mssql extension.
1. Select **Azure SQL Database**.
1. Enter the name **MyDBProj**, and press **Enter** to confirm.
1. Select the cloned GitHub repository folder to save the project.
1. For **SDK-style projects**, select **Yes (Recommended)**.
    > **Note:** Notice that a new project is created with the name **MyDBProj**.

### Create a new SQL file in the project

With the Azure SQL Database project created, let’s add a new SQL file to the project to create a new table.

1. On Visual Studio Code, select the **Database Projects** icon located in the activity bar on the left.
1. Right-click on your project name, and select **Add Table**.
1. Name the table **Employees** and press **Enter**.
1. Replace the existing script with the following code.

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. Close the editor. Notice that the `Employees.sql` file is saved in the project.

## Commit the changes to the repository

With the Azure SQL Database project created and the table script added to the project, let’s commit the changes to the repository.

1. On Visual Studio Code, select the **Source Control** icon located in the activity bar on the left.
1. Enter the message *Created project and added a create table script*.
1. Select **Commit** to commit your changes.
1. Under the ellipsis, select **Push** to push the changes to the repository.

## Verify the changes in the repository

Now that you have pushed the changes, let's verify them in the GitHub repository.

1. Open the [GitHub](https://github.com) website.
1. Navigate to the **MyDBProj** repository.
1. On the **<> Code** tab, open the **MyDBProj** folder.
1. Check if the changes in the **Employees.sql** file are up to date.

## Set up Continuous Integration (CI) with GitHub Actions

GitHub Actions enable you to automate, customize, and run your software development workflows directly in your GitHub repository. In this section, you'll configure a GitHub Actions workflow to build and test your Azure SQL Database project by creating a new table in the database.

### Create a Service Principal

1. Select the **Cloud Shell** icon in the top-right corner of the Azure portal. It looks like a `>_` symbol. If prompted, choose **Bash** as the shell type.

1. Run the following command in the Cloud Shell terminal. Replace the values `<your_subscription_id>`, and `<your_resource_group_name>` with your actual values. You can get these values on the **Resource group** page on Azure portal.

    ```azurecli
    az ad sp create-for-rbac --name "MyDBProj" --role contributor --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group_name>
    ```

1. Copy the output to a text editor. We'll reference it in the next section.

### Add secrets to the repository

1. In the GitHub repository, select **Settings**.
1. Select **Secrets and variables**, and then **Actions**.
1. On the **Secrets** tab, select **New repository secret**, and provide the following information.

    | Name | Value |
    | --- | --- |
    | AZURE_CREDENTIALS | The service principal output copied in the previous section.|
    | AZURE_CONN_STRING | Your connection string. |

    Your Azure credentials should look similar to this:
   
   ```
   {
    "clientSecret": <your_service_principal_password>,
    "subscriptionId": <your_subscription_id>,
    "tenantId": <your_service_principal_tenant>,
    "clientId": <your_service_principal_appId>
    }
   ```
   
    Your connection string should look similar to this:

    ```Server=<your_sqldb_server>.database.windows.net;Initial Catalog=MyDB;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;Encrypt=True;Connection Timeout=30;```

### Create a GitHub Actions workflow

1. In the GitHub repository, select the **Actions** tab.
1. Select the **set up a workflow yourself** link.
1. Copy the code below on your **main.yml** file. The code includes the steps for building and deploying your database project.

    ```yaml
    name: Build and Deploy SQL Database Project
    on:
      push:
        branches:
          - main
    jobs:
      build:
        permissions:
          contents: 'read'
          id-token: 'write'
          
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: ${{ secrets.AZURE_CONN_STRING }}
              path: './MyDBProj/MyDBProj.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
      ```

      The **Build and Deploy SQL Project** step in your YAML file connects to your Azure SQL Database using the connection string stored in the `AZURE_CONN_STRING` secret. The action specifies the path to your SQL project file, sets the action to publish to deploy the project, and includes build arguments to compile in Release mode. Additionally, it uses the `/p:DropObjectsNotInSource=true` argument to ensure that any objects not present in the source are dropped from the target database during deployment.

1. Select the **Commit changes** button.
1. Select **Commit directly to the main branch**, and then **Commit changes** again.

### Test the GitHub Actions workflow

1. In the GitHub repository, select the **Actions** tab.
1. Select the **Build and Deploy SQL Database Project** workflow.
    > **Note:** You’ll see the workflow in progress. Wait for it to complete. If it has already finished, select the latest run to view the details.

### Verify the changes in the Azure SQL Database

With the GitHub Actions workflow set up to build and deploy your Azure SQL Database project, it’s time to verify the changes in your Azure SQL Database.

1. Sign in to the [Azure portal](https://portal.azure.com?azure-portal=true). 
1. Navigate to the **MyDB** SQL Database.
1. Select **Query editor**.
1. Connect to the database using the **sqladmin** credentials.
1. Under the **Tables** section, verify that the **Employees** table is created. Refresh if needed.

You have successfully set up a GitHub Actions workflow to build and deploy your Azure SQL Database project.

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in extra costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about the SQL Database Projects extension for Azure SQL Database, see [Getting started with the SQL Database Projects extension](https://learn.microsoft.com/azure-data-studio/extensions/sql-database-project-extension-getting-started?azure-portal=true).
