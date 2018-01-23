# Using YML and Object Parameters
The ARM templates my team was having to manage were starting to get out of control. They are chalked full of hard-to-read parameters and inconsistencies. The original intent of the ARM deployments was to have 1 ARM template to house all the resources that needed to be built in 1 repository, along with 1 parameters file to house all of the ARM parameters along with any build parameters needed. The downside of this is that the cluster of parameters mixed in with layers of resources can make viewing and managing the ARM template and parameters file pretty nasty.

The first step in refactoring the process was going to try and figure out how to better handle parameters between the parameter file and the ARM template. Like I said earlier, we use 1 parameter file to house all of our parameters for a deployment. This includes both `build` parameters and `ARM template` parameters. For **every** parameter you have in your parameters file, you have to implement the type and name of the parameter in your ARM template in order for it to build without error. This leads to a very unwieldy looking ARM template. One of the engineers I work with suggested that we pass an object to the ARM deploy rather than a file. In this way we would only have to declare the ARM specific parameters within the ARM template. This was a good idea. I took his suggestion and went one step further. I took out **all** the parameters in my ARM template and replaced it with 1 parameter, an object that held all of my parameters.

# Before

```
"parameters": {
    "siteName": {
      "type": "string"
    },
    "hostingPlanName": {
      "type": "string"
    },
    "siteLocation": {
      "type": "string"
    },
    "sku": {
      "type": "string"
    },
    "skuCode": {
      "type": "string"
    },
    "workerSize": {
      "type": "string"
    },
    "_artifactsLocationSasToken": {
      "type": "string"
    },
    "_artifactsLocation": {
      "type": "string"
    },
    "dbAccountName": {
      "type": "string"
    },
    "dbAccountTier": {
      "type": "string"
    },
    "subscriptionId": {
      "type": "string"
    },
    "tenantId": {
      "type": "string"
    },
    "resourceGroupName": {
      "type": "string"
    },
    "passId": {
      "type": "int"
    },
    "artifactUrl": {
      "type": "string"
    },
    "resourceGroupTags": {
      "type": "object"
    },
    "resourceGroupLocation": {
      "type": "string"
    }
  }

```

# After
```
"parameters": {
  "config": {
    "type": "secureObject"
  }
}
```

Passing an object (or a secureObject if you have have secrets in your object) gives you the leverage to include any parameters that one might think they need (or dont need) and inject them into the arm template without having to list out each parameter individually within the template.

Getting values from your object might look something like this:

```
[parameters('config').cosmosdb.template]
```

So now that I have a slightly cleaner looking ARM template, I have to figure out how to get the object into my build script to pass in everything I need.

The next step I took was to remove the use of the typical JSON parameters file we usually see in ARM template deploys, and replace it with a YML file. Although a JSON parameter file would work just as well, I thought the look and feel of the YML file was going to be easier to read and manage going forward.

# Before

```
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "siteName": {
            "value": "sampleapi-dev"
        },
        "hostingPlanName": {
            "value": "sampleapi-plan"
        },
        "siteLocation": {
            "value": "West US"
        },
        "sku": {
            "value": "Standard"
        },
        "workerSize": {
            "value": "0"
        },
        "skuCode": {
            "value": "S1"
        },
        "dbAccountName": {
            "value": "sampleapidb"
        },
        "dbAccountTier": {
            "value": "Standard"
        }
        "resourceGroupTags": {
            "value": {
                "object": [
                    {
                        "name": "application",
                        "value": "sampleapi"
                    },
                    {
                        "name": "cost-center",
                        "value": "xxxxxxxxxx"
                    },
                    {
                        "name": "environment",
                        "value": "dev"
                    },
                    {
                        "name": "managed-by",
                        "value": "Platforms Team"
                    }   
                ]
            }
        }
    }
}
```
# After

```
API:
  SiteName: 'sampleapi'
  SiteLocation: 'westus'
  ServicePlanSku: 'Standard'
  ServicePlanTier: 'S1'
  ServicePlanInstances: 1
CosmosDB:
  Exists: 'True'
  DBName: 'sampleapidb'
  DBTier: 'Standard'
Tags:
  Application: 'sampleapi'
  Cost-Center: 'xxxxxxxxxx'
  Environment: 'dev'
  Managed-By: 'Platforms Team'
```

I then used [@pscookiemonster](https://twitter.com/pscookiemonster?lang=en)'s [PSDeploy](https://github.com/RamblingCookieMonster/PSDeploy) module to help me figure out how to deserialize my yml file into a powershell hash table. In the PSDeploy module there is a private module called PSYaml (**not** the same as the PSGallery PSYaml module). This was the only module that I could find that would let me deserialize YML straight from a file. I did, however, have to make a minor tweak to get this module to perform correctly.

Now that I have a hash table in my build script that houses all of my parameters, I have a lot of flexibility to add or even change values based on any logic that I build. Here is a snippet from one of my scripts to show what that might look like:

```
    $config = Get-CscConfigFromYaml -Path $ConfigFilePath

    # Set optional parameters for ARM template
    $config.Add('buildNumber', $buildNumber)
    $config.API.Add('servicePlanName', "$webApiName-plan")
    $config.API.Add('stagingSlotName', $stagingSlotName)
    $config.API.Add('template', -join($envConfig.apiTemplate, $armContainerToken))
    $config.API.SiteName = $webApiName
    $config.CosmosDB.Add('template', -join($envConfig.cosmosDBTemplate, $armContainerToken))

    # Run ARM deployment
    $armParams = @{
        DeploymentDebugLogLevel = $DebugLevel
        Name = $buildNumber
        ResourceGroupName = $resourceGroupName
        TemplateFile = $templateFilePath
        Config = $config
    }
    New-AzureRmResourceGroupDeployment @armParams
```

So now that I have a more manageable process for defining my parameters, I can start to look at how to manage the resources within our ARM templates. [Part 2 coming soon...](https://github.com/TerrapinStation/blog/dynamicArmPart2.md)