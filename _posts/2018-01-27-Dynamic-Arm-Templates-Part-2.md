# Using Linked Templates with Conditional Logic

In [Part 1](https://terrapinstation.github.io/blog/2018/Dynamic-Arm-Templates-Part-1.html), I setup the ARM parameters file to be a YML file to increase the readability of parameters and I also greatly reduced the amount of parameters needed to define the ARM template by defining one object (or secureObject) that houses all of the needed parameters (some defined in the YML and some generated during the build script).

In Part 2, I will continue and try to make the ARM templates less chaotic by reducing the amount of resources needed to be present in the main template. This can be achieved by starting to incorporate linked templates in order to extract common, reusable resources that can be put into a standardized template. The example below shows how this can be done in regards to building a Web API with an associated CosmosDB database.

Here are the resources that exist in the main template currently:

```
  "resources": [
    {
      "apiVersion": "2015-04-08",
      "type": "Microsoft.DocumentDb/databaseAccounts",
      "name": "[parameters('dbAccountName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "name": "[parameters('dbAccountName')]",
        "databaseAccountOfferType": "[parameters('dbAccountTier')]",
        "locations": [
          {
            "locationName": "[resourceGroup().location]",
            "failoverPriority": 0
          }
        ]
      }
    },
    {
      "apiVersion": "2014-11-01",
      "name": "[parameters('hostingPlanName')]",
      "type": "Microsoft.Web/serverFarms",
      "location": "[parameters('siteLocation')]",
      "properties": {
        "name": "[parameters('hostingPlanName')]",
        "sku": "[parameters('sku')]",
        "workerSize": "[parameters('workerSize')]",
        "numberOfWorkers": 1
      }
    },
    {
      "apiVersion": "2014-04-01",
      "name": "[parameters('siteName')]",
      "type": "Microsoft.Insights/components",
      "location": "eastus",
      "tags": {
          "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('siteName'))]": "Resource",
          "displayName": "AppInsightsComponent"
      },
      "properties": {
          "applicationId": "[parameters('siteName')]"
      }
    },
    {
      "apiVersion": "2015-04-01",
      "name": "[parameters('siteName')]",
      "type": "Microsoft.Web/sites",
      "location": "[parameters('siteLocation')]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverFarms', parameters('hostingPlanName'))]",
        "[resourceId('microsoft.insights/components/', parameters('siteName'))]"
      ],
      "properties": {
        "serverFarmId": "[parameters('hostingPlanName')]"
      },
      "resources": [
        {
          "apiVersion": "2015-04-01",
          "name": "staging",
          "type": "slots",
          "location": "[parameters('siteLocation')]",
          "dependsOn": [
            "[resourceId('Microsoft.Web/Sites', parameters('siteName'))]"
          ],
          "properties": {
          },
          "resources": [
            {
              "name": "MSDeploy",
              "type": "extensions",
              "location": "[resourceGroup().location]",
              "apiVersion": "2015-08-01",
              "dependsOn": [
                "staging"
              ],
              "tags": {
                "displayName": "webDeploy"
              },
              "properties": {
                "packageUri": "[concat(parameters('_artifactsLocation'), parameters('_artifactsLocationSasToken'))]"
              }
            },
            {
              "apiVersion": "2015-04-01",
              "name": "appsettings",
              "type": "config",
              "dependsOn": [
                "staging",
                "MSDeploy"
              ],
              "properties": {
                "DbSettings:URL": "[variables('dbUrl')]",
                "DbSettings:Key": "[listKeys(resourceId('Microsoft.DocumentDb/databaseAccounts', parameters('dbAccountName')), '2015-04-08').primaryMasterKey]",
                "ApplicationInsights:InstrumentationKey": "[reference(concat('microsoft.insights/components/', parameters('siteName'))).InstrumentationKey]"
              }
            }            
          ]
        }
      ]
    }
  ]
```

So there are a few major problems that refactoring this arm template will aid in solving. One of the problems, mentioned earlier, is that the hodge-podge of resources in the template make it difficult to read what is actually being deployed, especially if someone is not accustomed to reading/building arm templates. The second problem is that if a team decides to standardize on how Web APIs or CosmosDBs are being built and if that standard would ever shift or change, the team would have to spend a significant amount of time refactoring all of the places these resources exist within various arm templates (which could be 100s of places). Multiply that effort for every other resource that you want to standardize, and the whole process turns into a management nightmare. The third problem is much like the second, which is that a team would have a difficult time enforcing standards on how they would want to build certain resources if there is no central place to manage the templates. For example, if part of the standard was to require an App Insights instance for every Web API that was being built, it would be easy for someone to strip that out of the above arm template and bypass the standard altogether.

I started to build a solution that I hoped would solve, or at least help address, some of these issues. The first step I took was to pull out any of the resources that were reusable and place them into their own arm templates. So I stripped out the cosmosDB section and put it into its own arm template. I then stripped out the remaining resources that were used to build the API and put them into their own arm template. I then built a new main template that linked to the standard cosmosDB and API templates. Here is what this looks like:

```
{
  "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "config": {
      "type": "secureObject"
    }
  },
  "variables": {
  },
  "resources": [
    {
      "apiVersion": "2017-05-10",
      "name": "[concat(parameters('config').buildNumber, '-api-template')]",
      "type": "Microsoft.Resources/deployments",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
           "uri":"[parameters('config').api.template]",
           "contentVersion":"1.0.0.0"
        },
        "parameters": {
           "config":{"value": "[parameters('config')]"}
         }
      }
    },
    {
      "apiVersion": "2017-05-10",
      "name": "[concat(parameters('config').buildNumber, '-cosmosdb-template')]",
      "type": "Microsoft.Resources/deployments",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
           "uri":"[parameters('config').cosmosdb.template]",
           "contentVersion":"1.0.0.0"
        },
        "parameters": {
           "config":{"value": "[parameters('config')]"}
         }
      }
    }
  ]
}
```

The new template, with both the object based parameter along with the use of linked templates, makes for a very easy to read template. One can easily see that the template takes a parameter that houses all of the config parameters it needs, and the deployment is creating two resources; a cosmosDB and a web API.

There are a few important things to note in regards to using linked templates.
1. Any parameters that are needed in a linked template **must** be defined in the main template and passed to the linked template for use. In the example above, the re-factored template has everything defined in the config object, so there is no need to worry about defining extra parameters at all. One simply needs to pass the config object around to all the linked templates, and as long as the linked templates are configured correctly to use the config object, they will grab all of the necessary parameters they need for a successful deployment.
2. The linked template uri must be accessible to the azure deployment process. One way this can be solved it to set up an azure storage container for linked templates to reside in. Then, during deployment append temporary SAS tokens to the uri for a template, so that the deployment process could access the template contained in the azure storage container temporarily.

Now, if I wanted to use the same main template for all my API deployments, but maybe not all deployments needed a paired cosmosDB, I can start to look at adding in some conditional logic. The template will look exactly the same, with one additional line of code:  
```
"condition": "[equals(parameters('config').cosmosdb.exists, 'True')]"
```
This logic gives the ability to define in the YML parameters file whether or not a cosmosDB needs to be deployed alongside the API. If the YML config states that `cosmosdb.exists` is `True`, it will build a cosmosdDB. Otherwise, it will skip over the resource completely. Here is what the resource logic as a whole looks like:
```
    {
      "condition": "[equals(parameters('config').cosmosdb.exists, 'True')]"
      "apiVersion": "2017-05-10",
      "name": "[concat(parameters('config').buildNumber, '-api-template')]",
      "type": "Microsoft.Resources/deployments",
      "properties": {
        "mode": "Incremental",
        "templateLink": {
           "uri":"[parameters('config').api.template]",
           "contentVersion":"1.0.0.0"
        },
        "parameters": {
           "config":{"value": "[parameters('config')]"}
         }
      }
    }
```

The refactor appears to have solved, or at least greatly increased the readability of the main arm template. Secondly, one can now effectively standardize the deployment for reusable resources, and have a much better mechanism for updating the standard templates. One very important thing to note, is that if all API deployments are now referencing the one linked template, any changes to that template will effect **all** deployments. This is a very powerful and useful topology, but can also have unwanted changes due to an erroneous update or an update that was not well thought out. In my next blog post, I will be going into detail on how to minimize some of the danger of using linked templates by incorporating CI/CD processes for deploying these standardized linked templates in a safe manner.