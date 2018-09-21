# Build a real-time serverless chat app with Azure Functions

## Introduction

### Technologies used

* [Azure Storage](https://azure.microsoft.com/services/storage/?WT.mc_id=serverlesschatlab-tutorial-antchu) - Host the static website for the chat client UI
* [Azure Functions](https://azure.microsoft.com/services/functions/?WT.mc_id=serverlesschatlab-tutorial-antchu) - Backend API for creating and retrieving chat messages
* [Azure Cosmos DB](https://azure.microsoft.com/services/cosmos-db/?WT.mc_id=serverlesschatlab-tutorial-antchu) - Store chat messages
* [Azure SignalR Service](https://azure.microsoft.com/services/signalr-service/?WT.mc_id=serverlesschatlab-tutorial-antchu) - Broadcast new messages to connected chat clients

### Prerequisites

> If you are using a prepared virtual machine for this lab, all prerequisites should already be installed.

The following software is required to build this tutorial.

* [Git](https://git-scm.com/downloads)
* [Node.js](https://nodejs.org/en/download/) (Version 10.x)
* [.NET SDK](https://www.microsoft.com/net/download?WT.mc_id=serverlesschatlab-tutorial-antchu) (Version 2.x, required for Functions extensions)
* [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools) (Version 2)
* [Visual Studio Code](https://code.visualstudio.com/?WT.mc_id=serverlesschatlab-tutorial-antchu) (VS Code) with the following extensions
    * [Azure Functions](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions&WT.mc_id=serverlesschatlab-tutorial-antchu)
    * [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer&WT.mc_id=serverlesschatlab-tutorial-antchu)
* [Postman](https://www.getpostman.com/) (optional, for testing Azure Functions)


===


## Create Azure resources

You will build and test the Azure Functions app locally. The app will access some services in Azure that need to be created ahead of time.

### Log into the Azure portal

1. Go to the [Azure portal](https://portal.azure.com/) and log in with your credentials.

### Create an Azure Cosmos DB account

1. Click on the **Create a resource** (**+**) button for creating a new Azure resource.

1. Under **Databases**, select **Azure Cosmos DB**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | ID | A unique name for the Cosmos DB account |
    | API | SQL |
    | Resource group | New - enter a new resource group name |
    | Location | Select a location close to you |
    
    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/create-cosmosdb-screenshot.png)

1. Click **Create**.


### Create an Azure SignalR Service instance

1. Click on the **Create a resource** (**+**) button for creating a new Azure resource.

1. Search for **SignalR Service** and select it. Click **Create**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | Resource name | A unique name for the SignalR Service instance |
    | Resource group | Select the same resource group as the Cosmos DB account |
    | Location | Select a location close to you |
    | Pricing Tier | Free |

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/create-signalr-screenshot.png)
    
1. Click **Create**.


===


## Initialize the function app

### Create a new Azure Functions project

1. In a new VS Code window, use `File > Open Folder` in the menu to create and open an empty folder in an appropriate location. This will be the main project folder for the application that you will build.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-new-folder-screenshot.png)

1. Using the Azure Functions extension in VS Code, initialize a Function app in the main project folder.
    1. Open the Command Palette in VS Code by selecting `View > Command Palette` from the menu (shortcut `Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).
    1. Search for the "Azure Functions: Create New Project" command and select it.
    1. The main project folder should appear. Select it (or use "Browse" to locate it).
    1. In the prompt to choose a language, select **JavaScript**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-new-function-project-screenshot.png)


### Install function app extensions

This tutorial uses Azure Functions bindings to interact with Azure Cosmos DB and Azure SignalR Service. These bindings are available as extensions that need to be installed using the Azure Functions Core Tools CLI before they can be used.

1. Open a terminal in VS Code by selecting `View > Integrated Terminal` from the menu (Ctrl-`).

1. Ensure the main project folder is the current directory.

1. Install the Cosmos DB function app extension.
    ```
    func extensions install -p Microsoft.Azure.WebJobs.Extensions.CosmosDB -v 3.0.1
    ```

1. Install the SignalR Service function app extension.
    ```
    func extensions install -p AzureAdvocates.WebJobs.Extensions.SignalRService -v 0.3.0-alpha
    ```

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-install-func-extensions-screenshot.png)

### Configure application settings

When running and debugging the Azure Functions runtime locally, application settings are read from **local.settings.json**. Update this file with the connection strings of the Cosmos DB account and the SignalR Service instance that you created earlier.

1. In VS Code, select **local.settings.json** in the Explorer pane to open it.

1. Replace the file's contents with the following.
    ```nocopy
    {
        "IsEncrypted": false,
        "Values": {
            "AzureSignalRConnectionString": "<signalr-connection-string>",
            "AzureWebJobsCosmosDBConnectionString": "<cosmosdb-connection-string>",
            "WEBSITE_NODE_DEFAULT_VERSION": "10.0.0",
            "FUNCTIONS_WORKER_RUNTIME": "node"
        },
        "Host": {
            "LocalHttpPort": 7071,
            "CORS": "*"
        }
    }
    ```
    * Enter the Azure SignalR Service connection string into a setting named `AzureSignalRConnectionString`. Obtain the value from the **Keys** page in the Azure SignalR Service resource in the Azure portal; either than primary or secondary connection string can be used.
    * Enter the Azure Cosmos DB connection string into a setting named `AzureWebJobsCosmosDBConnectionString`. Obtain the value from the **Keys** page in the Cosmos DB account in the Azure portal; either than primary or secondary connection string can be used.
    * The `WEBSITE_NODE_DEFAULT_VERSION` setting is not used locally, but is required when deployed to Azure.
    * The `Host` section configures the port and CORS settings for the local Functions host.

1. Save the file.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-localsettings-screenshot.png)


===


## Create Azure Functions to save and retrieve chat messages

### CreateMessage function

#### Create the function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | HTTP Trigger |
    | Name | CreateMessage |
    | Authorization level | Anonymous |
    
    A folder named **CreateMessage** is created that contains the new function.

1. Open **CreateMessage/function.json** to configure bindings for the function. Modify the content of the file to the following.
    ```nocopy
    {
        "disabled": false,
        "bindings": [
            {
                "authLevel": "anonymous",
                "type": "httpTrigger",
                "direction": "in",
                "name": "req",
                "route": "messages",
                "methods": [
                    "post"
                ]
            },
            {
                "type": "http",
                "direction": "out",
                "name": "res"
            },
            {
                "name": "cosmosDBMessage",
                "type": "cosmosDB",
                "databaseName": "chat",
                "collectionName": "messages",
                "createIfNotExists": true, 
                "direction": "out",
                "connectionStringSetting": "AzureWebJobsCosmosDBConnectionString"
            }
        ]
    }
    ```
    This makes two changes to the function:
    * Changes the route to `messages` and restricts the HTTP trigger to the **POST** HTTP method.
    * Adds a Cosmos DB output binding that saves documents to a collection named `messages` in a database named `chat`.

1. Save the file.

1. Open **CreateMessage/index.js** to view the body of the function. Modify the content of the file to the following.
    ```nocopy
    module.exports = function (context, req) {  
        context.bindings.cosmosDBMessage = req.body;
        context.done();
    };
    ```
    This function takes the body from the HTTP request and saves it as a document in Azure Cosmos DB.

1. Save the file.


#### (Optional) Test the function

1. To run the function app locally, press **F5** in VS Code. If this is the first time, the Azure Functions host will start in the VS Code integrated terminal.

1. When the functions runtime is successfully started, the terminal output will display a URL for the local **CreateMessage** endpoint (by default, it is `http://localhost:7071/api/messages`).

1. Open Postman. Postman is an application to send HTTP requests.

1. Select `File > Import` from the menu.

1. Choose **Import from link** and paste in
    ```nocopy
    https://raw.githubusercontent.com/Azure-Samples/functions-serverless-chat-app-tutorial/master/requests/SignalRChat.postman_collection.json
    ```
    This loads a collection of HTTP requests for testing the function app locally. Click on the **Collections** tab in Postman to see it.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/postman-import-screenshot.png)

1. In the **SignalR Chat** collection, select the **Create a message** request.

1. Confirm the URL matches the one outputted by the function host and there is JSON message in the request body.

1. Click **Send**. The function app should return an HTTP status of 200.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/postman-test-createmessage-screenshot.png)

1. In the Azure portal, open the Cosmos DB account resource you created earlier.

1. Using the Cosmos DB Data Explorer, locate the **messages** collection in the **chat** database. The message sent from Postman should appear as a document in the collection.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/cosmosdb-data-explorer-screenshot.png)

1. Click the **Disconnect** button to disconnect the debugger from the function host.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-disconnect-functions-debug-screenshot.png)


### GetMessages function

#### Create the function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | HTTP Trigger |
    | Name | GetMessages |
    | Authorization level | Anonymous |
    
    A folder named **GetMessages** is created that contains the new function.

1. Open **GetMessages/function.json** to configure bindings for the function. Modify the content of the file to the following.
    ```nocopy
    {
        "disabled": false,
        "bindings": [
            {
                "authLevel": "anonymous",
                "type": "httpTrigger",
                "direction": "in",
                "name": "req",
                "route": "messages",
                "methods": [
                    "get"
                ]
            },
            {
                "type": "http",
                "direction": "out",
                "name": "res"
            },
            {
                "name": "messages",
                "type": "cosmosDB",
                "databaseName": "chat",
                "collectionName": "messages",
                "sqlQuery": "select * from c order by c._ts desc",
                "direction": "in",
                "connectionStringSetting": "AzureWebJobsCosmosDBConnectionString"
            }
        ]
    }
    ```
    This makes two changes to the function:
    * Changes the route to `messages` and restricts the HTTP trigger to the **GET** HTTP method.
    * Adds a Cosmos DB input binding that retrieves documents (in reverse chronological order) from a collection named `messages` in a database named `chat`.

1. Save the file.

1. Open **GetMessages/index.js** to view the body of the function. Modify the content of the file to the following.
    ```nocopy
    module.exports = function (context, req, messages) {
        context.res.body = messages;
        context.done();
    };
    ```
    This function takes the documents retrieved from Cosmos DB (chat messages in reverse chronological order) and returns them in the HTTP response body.

1. Save the file.


#### (Optional) Test the function

1. To run the function app locally, press **F5** in VS Code. If this is the first time, the Azure Functions host will start in the VS Code integrated terminal.

1. When the functions runtime is successfully started, the terminal output will display URLs for the local **CreateMessage** and **GetMessages** endpoints (by default, they are `http://localhost:7071/api/messages`).

1. (See above to open Postman and import a collection) In Postman, in the **SignalR Chat** collection, select the **Get messages** request.

1. Confirm the URL matches the one outputted by the function host and the HTTP method is **GET**.

1. Click **Send**. The function app should return the messages from Cosmos DB.

1. Press the **Disconnect** button to disconnect the debugger from the function host.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/postman-test-getmessages-screenshot.png)


===


## Create and run the chat client web user interface

The chat application's UI is a simple single page application (SPA) created with Vue JavaScript framework. It will be hosted separately from the function app. Locally, you will run the web interface using the Live Server VS Code extension.

1. In VS Code, create a new folder named **content** at the root of the main project folder.

1. In the **content** folder, create a new file named **index.html**.

1. Copy and paste the content of **[index.html](https://raw.githubusercontent.com/Azure-Samples/functions-serverless-chat-app-tutorial/master/snippets/index.html)**.

1. Save the file.

1. Press **F5** to run the function app locally (it may already be running) and attach a debugger.

1. With **index.html** open, start Live Server by opening the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`) and selecting **Live Server: Open with Live Server**. Live Server will open the application in a browser.

1. When the application prompts for a username, enter one. If you tested the function earlier, messages from your testing session will appear.

1. Enter a message in the chat box and press enter. Refresh the application to see new messages.

Next, you will integrate Azure SignalR Service into your application to allow messages to appear in real-time.


===


## Display real-time messages with Azure SignalR Service

Azure SignalR Service provides real-time messaging capabilities to supported clients, including web browsers. You will use Azure Functions to integrate with SignalR Service to broadcast new chat messages in real-time to connected browsers.

> The SignalR Service binding extension in Azure Functions is currently a community-supported project.


### SignalRInfo function

#### Create the function

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Create Function** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Function app folder | select the main project folder |
    | Template | HTTP Trigger |
    | Name | SignalRInfo |
    | Authorization level | Anonymous |
    
    A folder named **SignalRInfo** is created that contains the new function.

1. Open **SignalRInfo/function.json** to configure bindings for the function. Modify the content of the file to the following. This adds an input binding that generates valid credentials for a client to connect to an Azure SignalR Service hub named `chat`.
    ```nocopy
    {
        "disabled": false,
        "bindings": [
            {
                "authLevel": "anonymous",
                "type": "httpTrigger",
                "direction": "in",
                "name": "req"
            },
            {
                "type": "http",
                "direction": "out",
                "name": "res"
            },
            {
                "type": "signalRConnectionInfo",
                "name": "connectionInfo",
                "hubName": "chat",
                "direction": "in"
            }
        ]
    }
    ```

1. Open **SignalRInfo/index.js** to view the body of the function. Modify the content of the file to the following.

    ```nocopy
    module.exports = function (context, req, connectionInfo) {
        context.res = { body: connectionInfo };
        context.done();
    };
    ```

    This function takes the SignalR connection information from the input binding and returns it to the client in the HTTP response body.


#### (Optional) Test the function

1. To run the function app locally, press **F5** in VS Code. If this is the first time, the Azure Functions host will start in the VS Code integrated terminal.

1. When the functions runtime is successfully started, the terminal output will display URLs for the local endpoints, including **SignalRInfo** (by default, they are `http://localhost:7071/api/SignalRInfo`).

1. (See above to open Postman and import a collection) In Postman, in the **SignalR Chat** collection, select the **Get SignalR info** request.

1. Confirm the URL matches the one outputted by the function host and the HTTP method is **GET**.

1. Click **Send**. The function app should return connection information for SignalR Service.

1. Press the **Disconnect** button to disconnect the debugger from the function host.


### Broadcast new messages to all clients

#### Update the CreateMessage function

1. Open **CreateMessage/function.json** to configure bindings for the function. Add a `signalR` output binding by replacing the file's contents with the following.
    ```nocopy
    {
        "disabled": false,
        "bindings": [
            {
                "authLevel": "anonymous",
                "type": "httpTrigger",
                "direction": "in",
                "name": "req",
                "route": "messages",
                "methods": [
                    "post"
                ]
            },
            {
                "type": "http",
                "direction": "out",
                "name": "res"
            },
            {
                "name": "cosmosDBMessage",
                "type": "cosmosDB",
                "databaseName": "chat",
                "collectionName": "messages",
                "createIfNotExists": true, 
                "direction": "out",
                "connectionStringSetting": "AzureWebJobsCosmosDBConnectionString"
            },
            {
                "type": "signalR",
                "name": "signalRMessages",
                "hubName": "chat",
                "direction": "out"
            }
        ]
    }
    ```

1. Open **CreateMessage/index.js** to view the body of the function. Modify the content of the file to the following. This adds a line to output new chat messages to all clients connected to the SignalR Service hub.

    ```nocopy
    module.exports = function (context, req) {  
        context.bindings.cosmosDBMessage = req.body;
        context.bindings.signalRMessages = [{
            "target": "newMessage",
            "arguments": [req.body]
        }];
        context.done();
    };
    ```

#### Test the app

1. Press **F5** to run the function app locally (it may already be running) and attach a debugger.

1. With **index.html** open, start Live Server by opening the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`) and selecting **Live Server: Open with Live Server**. Live Server will open the application in a browser.

1. Now, new messages will appear as soon as they are sent. Open the app in more than one browser to see the real-time capabilities in action.


===


## Deploy to Azure

You have been running the function app and chat application locally. You will now deploy them to Azure.


### Log into Azure with VS Code

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure: Sign in** command.

1. Follow the instructions to complete the sign in process in your browser.


### Deploy function app

1. Select the Azure icon on the VS Code activity bar (left side).

1. Hover your mouse over the **Functions** pane and click the **Deploy to Function App** button.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Folder to deploy | Select the main project folder |
    | Subscription | Select your subscription |
    | Function app | Select **Create New Function App** |
    | Function app name | Enter a unique name |
    | Resource group | Select the same resource group as the Cosmos DB account and SignalR Service instance |
    | Storage account | Select **Create new storage account** |
    | Storage account name | Enter a unique name (3-24 characters, alphanumeric only) |
    | Location | Select a location close to you |
    
    A new function app is created in Azure and the deployment begins.

1. If prompted to switch the function app version from *latest* to *beta*, select **Update remote runtime**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-update-function-version-screenshot.png)


### Upload function app local settings

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Upload local settings** command.

1. When prompted, provide the following information.

    | Name | Value |
    |---|---|
    | Local settings file | local.settings.json |
    | Subscription | Select your subscription |
    | Function app | Select the previously deployed function app |
    | Function app name | Enter a unique name |

Local settings are uploaded to the function app in Azure. If prompted to overwrite existing settings, select **Yes to all**.

![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-update-function-settings-screenshot.png)


### Enable function app cross origin resource sharing (CORS)

Although there is a CORS setting in **local.settings.json**, it is not propagated to the function app in Azure. You need to set it separately.

1. Open the VS Code command palette (`Ctrl-Shift-P`, macOS: `Cmd-Shift-P`).

1. Search for and select the **Azure Functions: Open in portal** command.

1. Select the subscription and function app name to open the function app in the Azure portal.

1. Under the **Platform features** tab, select **CORS**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/functions-platform-features-screenshot.png)


1. Add an entry with the value `*`.

1. Remove all other existing entries.

1. Click **Save** to persist the CORS settings.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/functions-cors-screenshot.png)

> In a real-world application, instead of allowing CORS on all domains (`*`), a more secure approach is to enter specific CORS entries for each domains that requires it.

### Update function app URL in chat UI

1. In the Azure portal, navigate to the function app's overview page.

1. Copy the function app's URL.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/functions-get-url-screenshot.png)

1. In VS Code, open **index.html** and replace the value of `window.apiBaseUrl` with the function app's URL.

1. Save the file.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/vscode-paste-function-url-screenshot.png)


### Deploy web UI to blob storage

The web UI will be hosted using Azure Blob Storage's static websites feature.

1. Click on the **New** (**+**) button for creating a new Azure resource.

1. Under **Storage**, select **Storage account**.

1. Enter the following information.

    | Name | Value |
    |---|---|
    | Name | A unique name for the blob storage account |
    | Deployment model | Resource manager |
    | Account kind | StorageV2 (general purpose V2) |
    | Location | Select the same region as your other resources |
    | Replication | Locally-redundant storage (LRS) |
    | Performance | Standard |
    | Access tier | Hot |
    | Secure transfer required | Enabled |
    | Resource group | Select the same resource group as the Cosmos DB account |
    
    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/create-storage-screenshot.png)

1. Click **Create**.

1. When the storage account is created, open it in the Azure portal.

1. Select **Static website (preview)** in the left navigation.

1. Select **Enable**.

1. Enter `index.html` as the **Index document name**.

1. Click **Save**.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/storage-enable-static-websites-screenshot.png)

1. Click on the **$web** link on the page to open the blob container.

1. Click **Upload** and upload all the files in the **content** folder.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/storage-upload-screenshot.png)

1. Go back to the **Static website** page. Copy the **Primary endpoint** address and open it in a browser.

    ![](https://github.com/Azure-Samples/functions-serverless-chat-app-tutorial/raw/master/media/storage-primary-endpoint-screenshot.png)

The chat application will appear. Congratulations on creating and deploying a serverless chat application to Azure!
