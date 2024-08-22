---
lab:
    title: 'Configure and Deploy CI/CD Pipelines for Azure SQL Database Projects'
    module: 'Develop for an Azure SQL Database'
---

# Configure and deploy CI/CD pipelines for Azure SQL Database projects

In this exercise you will create, configure, and deploy CI/CD pipelines for Azure SQL Database projects using Visual Studio Code and GitHub Actions. This will allow you to familiarize yourself with the process of setting up CI/CD pipelines for Azure SQL Database projects.

## Before you start

Before you can start this exercise, you will need:

- An Azure subscription with appropriate permissions to create and manage resources.
- [Visual Studio Code](https://code.visualstudio.com/download) installed on your computer with the following extensions:
  - [SQL Database Projects](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql).
  - [GitHub Pull Requests](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github).
- GitHub accounts.
- Basic knowledge of *GitHub Actions* pipelines.

## Set up the environment

There are a few steps you need to take to set up the environment for this exercise.

### Create an Azure SQL Database

First, you need to create a new Azure SQL Database.

1. On the Azure portal, navigate to the **Azure SQL** page.
1. Select **Create**.
1. Select **SQL Database**, *Single database* and the **Create** button.
1. Fill in the required information on the **Create SQL Database** dialog and select **OK** (leave all other options to their default).

    | Setting | Value |
    | --- | --- |
    | Free serverless offer | *Apply offer* |
    | Subscription | Your subscription |
    | Resource group | *Select or create a new resource group* |
    | Database name | *dp3020db02* |
    | Server | *Create a new server* |
    | Server admin login | *Select a unique name* |
    | Location | *Select a location* |
    | Authentication method | *Use SQL authentication* |
    | Server admin login | *dp3020admin* |
    | Password | *Enter a password* |
    | Confirm password | *Confirm the password* |

1. Select **Review + create** and then **Create**.
1. Once the deployment is complete, navigate to the Azure SQL Database *server* you created.
1. Go to the **Security** and then **Networking** settings and add your IP address to the firewall rules.
1. Select the **Allow Azure services and resources to access this server** option. This option allows GitHub Actions to access the database.

    [!NOTE] In a production environment, you should restrict access to only the required IP addresses. Additionally you might want to use Managed Identities to access the database from your GitHub Action instead of using SQL authentication. For more information, see the [sql-action connection guide](https://https://github.com/Azure/sql-action/blob/master/CONNECTION.md).

1. Select **Save**.

You have successfully created an Azure SQL Database.

### Set up a GitHub repository

Next, you need to set up a new GitHub repository.

1. Open the [GitHub website](https://github.com).
1. Sign in to your GitHub account.
1. Under your account, select **Repositories**.
1. Select **New**.
1. Under ***Owner***, *select your account*/under ***Repository name***, enter the name **dp3020ex02**.
1. Make the repository **Private**.
1. Select **Create repository**.

### Install the Visual Studio Code extensions and clone the repository

Before you can clone the repository, you need to install the required Visual Studio Code extensions.

1. Open Visual Studio Code.
1. Install the **SQL Database Projects** extension.
1. Install **GitHub Pull Requests** extension. If you already have GitHub installed in your computer, you can skip this step.

Now you can clone the repository you created in the previous step.

1. In Visual Studio Code, select **View** > **Command Palette**.
1. In the command palette, type `Git: Clone` and select it.
1. Enter the URL of the repository you created in the previous step and select **Clone**.

## Create and configure an Azure SQL Database project

An Azure SQL Database project is a Visual Studio project type that allows you to develop, build, test, and publish your database schema and data. In this exercise, you will create an Azure SQL Database project and configure it to connect to the Azure SQL Database you created earlier.

1. In Visual Studio Code, select **View** > **Command Palette**.
1. In the command palette, type `Database projects: New` and select it.
1. Select **Azure SQL Database**.
1. Enter the name **dp3020db02** and select **Enter**.
1. Select the cloned GitHub repository folder to save the project.
1. Under SDK-style project, select **Yes**.
1. You will notice that a new project is created with the name **dp3020db02**.

### Create a new SQL file in the project

Now that you have created the Azure SQL Database project, let's add a new SQL file to the project to create a new table.

1. In Visual Studio Code activity bar, select the **Database Projects** icon.
1. Right-click on your project name, and select **Add Table**.
1. Name the table **Employees** and select **Enter**.
1. Replace the script with the following code:

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. Save the file. Notices how the file is saved as `Employees.sql` in the project.

### Commit the changes to the repository

Now that you have created the Azure SQL Database project and added a new SQL file to the project, let's commit the changes to the repository.

1. In Visual Studio Code, select the **Source Control** icon.
1. Enter the message `Created project and added a create table script`.
1. Right-click on **Changes** and select **Stage All Changes**.
1. Select the **Commit** icon to commit the changes.
1. Under the ellipsis, select **Push** to push the changes to the repository.

### Verify the changes in the repository

Now that you have pushed the changes to the repository, let's verify the changes are in the GitHub repository.

1. Open the GitHub website.
1. Navigate to the repository you created earlier.
1. Verify that the changes you made are under the **Code** tab, refresh if needed.

## Set Up Continuous Integration with GitHub Actions

GitHub Actions allows you to automate, customize, and execute your software development workflows right in your GitHub repository. In this exercise, you will set up a GitHub Actions workflow to build and test your Azure SQL Database project by creating a new table in the database.

### Add secrets to the repository

1. In the GitHub repository, select **Settings**.
1. Select **Secrets and variables**.
1. Select **Actions**.
1. Under the **Variables** tab, select to add **New repository secrets**, and add the following secrets:

    | Name | Value |
    | --- | --- |
    | SQL_SERVER_NAME | *Your Azure SQL Database server name* |
    | SQL_ADMIN_USER | ***dp3020admin***** |
    | SQL_ADMIN_PASSWORD | *Your Azure SQL Database admin password* |

### Create a GitHub Actions workflow

1. In the GitHub repository, select the **Actions** tab.
1. Select **Set up a workflow yourself**.
1. Modify the YAML file to include steps for building and deploying your database project:

    ```yaml
    name: Build and Deploy SQL Database Project
    
    on:
      push:
        branches:
          - main
    
    jobs:
      build:
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
    
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3
    
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: "Server=tcp:${{ secrets.SQL_SERVER_NAME }}.database.windows.net,1433;Initial Catalog=dp3020db02;Persist Security Info=False;User ID=${{ secrets.SQL_ADMIN_USER }};Password=${{ secrets.SQL_ADMIN_PASSWORD }};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
              path: './dp3020db02/dp3020db02.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
        ```

1. Select the **Commit changes** button.
1. Select **Commit directly to the main branch** and then **Commit changes** again.

### Test the GitHub Actions workflow

1. In the GitHub repository, select the **Actions** tab.
1. Select the **Build and Deploy SQL Database Project** workflow.
1. You will see the workflow running. Wait for the workflow to complete. If the workflow already completed, select the latest run to view the details.

### Verify the changes in the Azure SQL Database

Now that you have set up the GitHub Actions workflow to build and deploy your Azure SQL Database project, let's verify the changes in the Azure SQL Database.

1. Open the Azure portal.
1. Navigate to the Azure SQL Database you created earlier.
1. Select **Query editor**.
1. Connect to the database using the **dp3020admin** credentials.
1. Under the Tables section, verify that the **Employees** table is created. Refresh if needed.

You have successfully set up a GitHub Actions workflow to build and deploy your Azure SQL Database project.

## Clean up

After you have completed this exercise, you can clean up the resources you created to avoid incurring additional costs.

1. Delete the Azure SQL Database you created.
1. Delete the GitHub repository you created.
1. Uninstall the Visual Studio Code extensions you installed if you no longer need them.
