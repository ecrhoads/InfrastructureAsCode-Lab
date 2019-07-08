# Infrastructure as Code Lab

The purpose of this lab is to walk you through creating a terraform module which will deploy the necessary infrastructure for an Azure-based web application. At the end of this lab, you will be able to reach the application on a public url and perform basic interaction.

## Getting Started

If you have Terraform, Visual Studio Code, Visual Studio 2019, and/or Azure CLI already on your workstation, you can skip these sections.

**Terraform Setup**
1. Download and Unzip the Terraform package for your OS: https://www.terraform.io/downloads.html
    -NOTE: The Terraform binary should be unzipped to a root folder location where you intend to perform work on your laptop (C:\Temp\Terraform as an example)
2. Navigate to: Control Panel > System > System Settings > Environment Variables
3. Click edit on the PATH variable.
4. Include the path where the terraform binary was unzipped.
5. To verify installation, open a new terminal and type 'terraform'. You should see usage and commands available to run.

**Visual Studio Code Setup**
1. Download the Visual Studio Code package for your OS: https://code.visualstudio.com/download
2. Run the installation (VSCodeUserSetup-{version}.exe).
3. Open Visual Studio Code, then navigate to: View > Extensions.
4. In the search bar type: Terraform then click Install.
5. To verify installation, make sure the Terraform icon is visible in VS Code.

**Visual Studio 2019 Community Edition Setup**
1. Download the Visual Studio 2019 Community Edition from: https://visualstudio.microsoft.com/downloads/
2. Run the installation.
3. Include the "Azure development" workload and Microsoft Azure SDK for .NET

**Azure CLI Setup**
1. Download the Azure CLI for your OS: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
2. Run the installation. 
3. To verify installation, open a terminal and type `az login`. When prompted enter your KiZAN credentials.

**Obtain Azure Secrets**
1. Open a terminal and type `az login` (skip if already done). When prompted enter your KiZAN credentials.
2. Type `az account show`. Ensure the "name" property is "KiZAN Production Azure".
3. Copy the "id" (This is the Subscription ID) and the "tenantId" property values to a text file. We will use these later in the lab.

## Module 1: Create a Terraform Module

In this section we are going to create a new terraform module which can be re-used to create identical application infrastructure for a web application using CosmosDB as a backend.

### 1.1: Setup the Local Repo
First, we need to setup the folder structure for our repository. We will include a modules folder and a 'dev' folder for our environment.

1. Open Visual Studio Code, then click File, Open Folder...
2. Select the Folder where the Terraform binary was unzipped.
3. Create a new folder called "_modules" by clicking the New Folder icon in the Explorer pane. Technically, every folder is a module in the eyes of Terraform. But this module is our "stamp" of an environment which will be generalized to be re-usable.
4. Create a sub folder within "_modules" called "cosmos_web_app".
5. Create another top-level folder called "dev".  
6. Within "_modules/cosmos_web_app/" click New File and name it "main.tf". This is where we will do the bulk of our work.
7. Repeat Step 6, but name the new file "vars.tf".
8. Within "dev/" click New File and name it "main.tf". Files in different folders can re-use names without any conflicts.

### 1.2: Add Terraform Resources
For our web application infrastructure "stamp" we are going to need 4 core resources in Azure. A Resource Group, an Application Service Plan, an Application Service, and a CosmosDB.

1. To Create a new resource group in the East US 2 location, open your "_modules/cosmos_web_app/main.tf file and paste the following code:

```
#Resource Group
resource "azurerm_resource_group" "app-rg" {
    name = "tf-todo-dev-rg"
    location = "east us 2"
}
```

This resource only requires two parameters, which we have filled in here. There are additional parameters which are accepted by terraform if we wanted to configure aspects like Tagging, etc. For this lab we will only be filling in required parameters. If you want to see the full list of parameters available for a resource, click on the highlighted resource name "azurerm_resource_group" in your VS Code main.tf file.

2. Next we are going to add the Application Service Plan by pasting the following code below the resource group resource:

```
resource "azurerm_app_service_plan" "standard_app_plan" {
    name = "tf-standard-plan"
    location = "${azurerm_resource_group.application_rg.location}"
    resource_group_name = "${azurerm_resource_group.application_rg.name}"
    sku {
        tier = "Basic"
        size = "B1"
    }
}
```

Notice the "location" and "resource_group_name" properties are using built-in syntax to obtain their values. We can pull outputs from other resources using the syntax "azurerm_<resource_type>.<resource_alias>.<property>". In this case the resource type is "resource_group", the resource alias is "application_rg" (We provided the resource alias in the previous code snippet next to the declaration of the resource type), and the property we care about for location is ".location" and for resource_group_name it's ".name".

3. Now that we have declared the Plan our Application Service will use, we need to create the Azure Web App. Use the following code:

```
resource "azurerm_app_service" "web_app_service" {
    name = "tf-todo-dev-app"
    location = "east us 2"
    resource_group_name = "${azurerm_resource_group.application_rg.name}"
    app_service_plan_id = "${azurerm_app_service_plan.app_plan.id}"
}
```

While we are placing resources in a logical order, Terraform is a declarative language. This means we could have added these resource code snippets in ANY order of the file. Terraform will figure out the right way to order resources for proper deployment.

4. Last, we want to deploy the CosmosDB instance. Use this code:

```
resource "azurerm_cosmosdb_account" "db" {
  name                = "tf-cosmos-db"
  location            = "east us 2"
  resource_group_name = "${azurerm_resource_group.rg.name}"
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  enable_automatic_failover = true

  consistency_policy {
    consistency_level       = "Session"
  }

  geo_location {
    location          = "east us 2"
    failover_priority = 0
  }
}
```

When parameters of a resource have multiple/sub-parameters, they are encapsulated with `{}`. In this case, CosmosDB has a multi-property parameter for consistency_policy and geo_location.

### 1.3: Generalize the module
We have created a module which will deploy a basic version of all the resources we need. BUT, we cannot re-use this module yet. There are certain parameters we have hard-coded which will prevent us from using this module in different ways. Let's change that by using variables.

1. Open the vars.tf file created earlier.
2. We are going to create 3 variables which will allow us to generalize the previously created module. Paste the following code into the vars.tf file:

```
variable "application_short_name" {
    description = "Abbreviated name for the application, used to name associated infrastructure resources"
}

variable "environment" {
    description = "The application environment (Dev, Test, QA, Prod)"
}

variable "location" {
    description = "The Azure Region where resources will be deployed"
}
```

You declare variables in Terraform by supplying "variable" and then a name for the variable followed by `{}`. If you want to include a default value you can specify `default = "value"`. In our case, all we are including are descriptions of how the variables are used.

3. Next, open the main.tf file in the "_modules/web_cosmos_app/" folder.
4. To generalize the resource group, we are going to change the name and location parameters to use the variables we just created. Replace the entire "azurerm_resource_group" block with the following code:

```
resource "azurerm_resource_group" "app-rg" {
    name = "tf-${var.application_short_name}-${var.environment}-rg"
    location = "${var.location}"
}
```

Variables can be used by specifying ${var.<name of variable>}. You can string multiple variables together to create unique values for things like naming of resources.

5. To generalize the Application Service Plan, we are going to change the location. We could add additional variables for the Tier and Size, but in this case, our "stamp" sets those values to a default. Since these values do not cause any conflicts we are not required to change them for re-usability. This shows that we can pick and choose what we want to expose for dynamic updating. Replace the entire "azurerm_app_service_plan" resource block with the following code:

```
resource "azurerm_app_service_plan" "standard_app_plan" {
    name = "tf-standard-plan"
    location = "${var.location}"
    resource_group_name = "${azurerm_resource_group.application_rg.name}"
    sku {
        tier = "Basic"
        size = "B1"
    }
}
```
6. To generalize the App Service, add the following code:

```
resource "azurerm_app_service" "web_app_service" {
    name = "tf-${var.application_short_name}-${var.environment}-app"
    location = "${var.location}"
    resource_group_name = "${azurerm_resource_group.application_rg.name}"
    app_service_plan_id = "${azurerm_app_service_plan.app_plan.id}"
}
```

7. To generalize the Cosmos DB, we actually need to create anothe resource first. We are going to grab a random integer to help with unique naming of the CosmosDB resource. To do so, we will use the "random_integer" resource. Paste the following code just above the Cosmos DB resource block:

```
resource "random_integer" "ri" {
  min = 10000
  max = 99999
}
```

8. Now, we will use the random integer resource from above and the variables we created to generalize the CosmosDB resource. Replace the entire "azurerm_cosmosdb_account" block with the following code:

```
resource "azurerm_cosmosdb_account" "db" {
  name                = "tf-cosmos-${random_integer.ri.result}-${var.environment}-db"
  location            = "${var.location}"
  resource_group_name = "${azurerm_resource_group.rg.name}"
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  enable_automatic_failover = true

  consistency_policy {
    consistency_level       = "Session"
  }

  geo_location {
    location          = "${var.location}"
    failover_priority = 0
  }
}
```

Notice how we obtain the random integer by supplying "${random_integer.ri.result}". This is a built-in resource in Terraform which will produce a random value between the declared min and max values. This is being used to ensure we have a unique namespace for our cosmosdb resource. 

## Module 2: Use the Terraform Module

The module is now usable. So let's use it!

1. Open the "dev/main.tf" file.
2. We need to declare our provider in this file. A provider is basically a plugin that tells Terraform what API is being used. Paste the following code into the main.tf file:

```
provider "azurerm" {
    tenant_id       = ""
    subscription_id = ""
}
```
Paste the Tenant ID and Subscription ID you copied into a text file at the beginning of this lab into the appropriate parameters. This tells Terraform what tenant and subscription resources will be deployed into.

3. Next, we are going to call the previously created module and supply the required values. Paste the following code below the provider block:

```
module "todo_app" {
    source = "../_modules/web_cosmos_app/"

    application_short_name = "todo"
    environment = "dev"
    location = "east us 2"
}
```

The name of the module can be anything you want. In our case the name is "todo_app". The source specifies where the module is located in the repository. This can also be an external source, but in our case it is a local one. The three parameters we are supplying information into are "application_short_name", "environment", and "location". Sound familiar? These are the variables we created earlier. Any variable without a default value must be supplied a value by the user when calling the module.

Now that we have the file ready to go, we need to deploy the resources.

4. Click "Terminal > New Terminal", then hit Enter.
5. We need to make sure we are in the "dev" folder. Run `cd ./dev` to move into the dev directory. If you are not in your working folder at all, append the cd to supply the entire folder path.
6. Login to Azure by typing `az login` and supplying your credentials.
7. Type `terraform init` and press enter. This tells Terraform to initialize the configuration files in this folder.
8. Next, let's see what our terraform will create. Type `terraform plan` and press enter.

If the code was written successfully, this should produce a list of all the resources terraform will create in Azure with values (or computed values) supplied. This is a way for us to check what we are deploying and make sure it is right before we actually deploy the resources to Azure.

9. Let's deploy! Type `terraform apply` and press enter. When prompted, type 'y' to confirm you want to deploy the resources.

This will take any files in the folder we have selected and run them through the terraform logic. In our case, we told Terraform to use the Azure API to deploy the resources in our custom module into our Tenant and Subscription. When the apply is complete, login to the portal to see the created resources.

We've created the infrastructure for an application, but how do we marry this with the development of an application? We are going to do this in the next section. But first, we need to copy two more values.

10. Click on the CosmosDB resource we created. Under Settings, click "Keys".
11. Copy the URI and the PRIMARY KEY values a notepad. We will use these later.

## Module 3: Create an ASP.NET MVC Application*

The description of this application setup will be brief, as it is only used to demonstrate how to bind both the infrastructure and development sides together.


### 3.1: Setup the Project

First, we are going to create a new ASP.NET MVC Project and add the CosmosDB libraries.

1. Open Visual Studio and Click Create a New Project
2. Search for "ASP.NET" and select the second option titled "ASP.NET Web Application (.NET Framework)"
3. Set the project name as "todo", specify a location to create the project and click OK.
4. Select MVC for the template and Click Create.
5. In the Solution Explorer, right click the project name "todo", then select "Manage NuGet Packages".
6. Search for "Microsoft.Azure.DocumentDB" and click to install this library for CosmosDB.

### 3.2: Setup the MVC Components

Here we are going to setup the models, views, and controllers for the web application.

First, the JSON data model:

1. In Solution Explorer, right-click Models, click Add, click Class, name the class Item.cs, then click Add.

2. In the Item.cs file, add after the last using statement `using Newtonsoft.Json;`

3. Replace the `public class Item` block with:

```
public class Item
 {
     [JsonProperty(PropertyName = "id")]
     public string Id { get; set; }

     [JsonProperty(PropertyName = "name")]
     public string Name { get; set; }

     [JsonProperty(PropertyName = "description")]
     public string Description { get; set; }

     [JsonProperty(PropertyName = "isComplete")]
     public bool Completed { get; set; }
 }
```

Next, the Controller:

4. In Solution Explorer, right-click Controllers folder, click Add, Controller, then select MVC 5 Controller - Empty and click Add.

5. Name the Controller "ItemController" then click Add.

Now we are going to create 3 Views (Item Index, New Item, and Edit Item) which will make up the webpage views:

6. In Solution Explorer, expand Views, right-click the empty Item folder, Add, View.

7. Perform the following in the Add View:

-Name: Index
-Template: List
-Model class: Item (todo.Models)
-Layout Page: ~/Views/Shared/_Layout.cshtml

8. Repeat step 7 for New Item view, replacing both the Name and Template values with "Create"

9. Repeat step 7 for Edit Item view, replacing both the Name and Template values with "Edit"

### 3.3: Wire CosmosDB into Code

1. In Solution Explorer, right-click the project, click Add, Class. Name the class "DocumentDBRepository" and click Add.

2. Add the following above the namespace declaration:

```
using Microsoft.Azure.Documents; 
using Microsoft.Azure.Documents.Client; 
using Microsoft.Azure.Documents.Linq; 
using System.Configuration;
using System.Linq.Expressions;
using System.Threading.Tasks;
using System.Net;
```

3. Replace the `public class DocumentDBRepository` block with:

```
public static class DocumentDBRepository<T> where T : class
 {
     private static readonly string DatabaseId = ConfigurationManager.AppSettings["database"];
     private static readonly string CollectionId = ConfigurationManager.AppSettings["collection"];
     private static DocumentClient client;

     public static void Initialize()
     {
         client = new DocumentClient(new Uri(ConfigurationManager.AppSettings["endpoint"]), ConfigurationManager.AppSettings["authKey"]);
         CreateDatabaseIfNotExistsAsync().Wait();
         CreateCollectionIfNotExistsAsync().Wait();
     }

     private static async Task CreateDatabaseIfNotExistsAsync()
     {
         try
         {
             await client.ReadDatabaseAsync(UriFactory.CreateDatabaseUri(DatabaseId));
         }
         catch (DocumentClientException e)
         {
             if (e.StatusCode == System.Net.HttpStatusCode.NotFound)
             {
                 await client.CreateDatabaseAsync(new Database { Id = DatabaseId });
             }
             else
             {
                 throw;
             }
         }
     }

     private static async Task CreateCollectionIfNotExistsAsync()
     {
         try
         {
             await client.ReadDocumentCollectionAsync(UriFactory.CreateDocumentCollectionUri(DatabaseId, CollectionId));
         }
         catch (DocumentClientException e)
         {
             if (e.StatusCode == System.Net.HttpStatusCode.NotFound)
             {
                 await client.CreateDocumentCollectionAsync(
                     UriFactory.CreateDatabaseUri(DatabaseId),
                     new DocumentCollection { Id = CollectionId },
                     new RequestOptions { OfferThroughput = 1000 });
             }
             else
             {
                 throw;
             }
         }
     }
 }
 ```

 4. We are reading values from configuration, so open the main Web.config file and add the following lines under the <AppSettings> section:

 ```
 <add key="endpoint" value="enter the URI from the Keys blade of the Azure Portal"/>
 <add key="authKey" value="enter the PRIMARY KEY, or the SECONDARY KEY, from the Keys blade of the Azure  Portal"/>
 <add key="database" value="ToDoList"/>
 <add key="collection" value="Items"/>
 ```

 5. Update the "endpoint" with the CosmosDB URI copied earlier in this lab, and update the "authKey" with the CosmosDB Primary Key copied earlier in this lab.

 6. We are going to add functionality to display incomplete items in the 'todo list application'. Copy and paste the following code anywhere within the DocumentDBRepository class:

 ```
 public static async Task<IEnumerable<T>> GetItemsAsync(Expression<Func<T, bool>> predicate)
 {
     IDocumentQuery<T> query = client.CreateDocumentQuery<T>(
         UriFactory.CreateDocumentCollectionUri(DatabaseId, CollectionId))
         .Where(predicate)
         .AsDocumentQuery();

     List<T> results = new List<T>();
     while (query.HasMoreResults)
     {
         results.AddRange(await query.ExecuteNextAsync<T>());
     }

     return results;
 }
 ```

 7. Open the ItemController we created earlier and add the following using statements above the namespace declaration:

```
using System.Net;
 using System.Threading.Tasks;
 using todo.Models;
```

8. Replace:
```
//GET: Item
 public ActionResult Index()
 {
     return View();
 }
```

With:
```
[ActionName("Index")]
 public async Task<ActionResult> IndexAsync()
 {
     var items = await DocumentDBRepository<Item>.GetItemsAsync(d => !d.Completed);
     return View(items);
 }
```

9. Open Global.asax.cs and add the following line to the Application_Start method: `DocumentDBRepository<todo.Models.Item>.Initialize();`

10. Open App_Start\RouteConfig.cs and locate the line starting with "defaults:" and change it to: `defaults: new { controller = "Item", action = "Index", id = UrlParameter.Optional }`. This tells the application to use Item as the controller and Index as the view.

Now we are going to add functionality to add items into our database. 

11. In the DocumentDBRepository class add:
```
public static async Task<Document> CreateItemAsync(T item)
{
    return await client.CreateDocumentAsync(UriFactory.CreateDocumentCollectionUri(DatabaseId, CollectionId), item);
}
```
This method takes an object passed to it and persists it in CosmosDB.

12. Open ItemController.cs and add the following code:
```
[ActionName("Create")]
 public async Task<ActionResult> CreateAsync()
 {
     return View();
 }
```

This is how the application knows what to do for the Create action.

13. Add the next block of code to the ItemController.cs class to tell the application what to do with a form POST for this controller:

```
[HttpPost]
 [ActionName("Create")]
 [ValidateAntiForgeryToken]
 public async Task<ActionResult> CreateAsync([Bind(Include = "Id,Name,Description,Completed")] Item item)
 {
     if (ModelState.IsValid)
     {
         await DocumentDBRepository<Item>.CreateItemAsync(item);
         return RedirectToAction("Index");
     }

     return View(item);
 }
```
This code calls into the DocumentDBRepository and uses the CreateItemAsync method to persist the new todo item to the database.

There is one last thing to do. We need to add the ability to edit Items in the database and to mark them as complete. We already created the view, so we just need to add code to the controller and DocumentDBRepository class again.

14. Add the following to the DocumentDBRepository class:

```
public static async Task<Document> UpdateItemAsync(string id, T item)
 {
     return await client.ReplaceDocumentAsync(UriFactory.CreateDocumentUri(DatabaseId, CollectionId, id), item);
 }

 public static async Task<T> GetItemAsync(string id)
 {
     try
     {
         Document document = await client.ReadDocumentAsync(UriFactory.CreateDocumentUri(DatabaseId, CollectionId, id));
         return (T)(dynamic)document;
     }
     catch (DocumentClientException e)
     {
         if (e.StatusCode == HttpStatusCode.NotFound)
         {
             return null;
         }
         else
         {
             throw;
         }
     }
 }
 ```

 15. Add the following to the ItemController class:

```
[HttpPost]
 [ActionName("Edit")]
 [ValidateAntiForgeryToken]
 public async Task<ActionResult> EditAsync([Bind(Include = "Id,Name,Description,Completed")] Item item)
 {
     if (ModelState.IsValid)
     {
         await DocumentDBRepository<Item>.UpdateItemAsync(item.Id, item);
         return RedirectToAction("Index");
     }

     return View(item);
 }

 [ActionName("Edit")]
 public async Task<ActionResult> EditAsync(string id)
 {
     if (id == null)
     {
         return new HttpStatusCodeResult(HttpStatusCode.BadRequest);
     }

     Item item = await DocumentDBRepository<Item>.GetItemAsync(id);
     if (item == null)
     {
         return HttpNotFound();
     }

     return View(item);
 }
```

### 3.4: Test and Deploy the Application

We are going to build, test, and then deploy the application we created in the previous sections.

1. From within Visual Studio, hit F5 to build the application in debug mode.
2. Click "Create New" and add any values for Name and Description. Leave the completed checkbox empty. Click Create. Create a couple items.
3. Click "Edit" next to an item and mark it as complete.
4. Now that we have validated, click Ctrl+F5 to stop debugging the app.
5. To publish the application to Azure, in Solution Explorer, right-click on the project "todo" and click Publish.
6. Click "Microsoft Azure App Service" and choose "Select Existing".
7. Ensure the right Subscription is selected, then choose the Resource Group and App Service we created earlier in this lab. Click Create.
8. Visit your new web application backed by cosmos db by navigating to the FQDN of the azure web app that is created by default.

This concludes the Infrastructure as Code Lab.

*The ASP.NET MVC Application Configuration Instructions used here to illustrate how Infrastructure as Code and Development can be integrated together, were taken from this Microsoft tutorial: https://docs.microsoft.com/en-us/azure/cosmos-db/sql-api-dotnet-application.
