# Setting up CI/CD for Reusable Arm Templates

In my previous posts, I went into detail on how to create dynamic arm templates. One of the key features of making your arm templates dynamic is to leverage the use of linked templates. Linked templates allow a team to have a mechanism to update/manage how resources are being created without having to touch every deployment pipeline that deploys similar resources. The perceived downside of this is that if someone was to make an erroneous change to a linked template, then all deploys that reference that linked template will essentially be in an erroneous state. In order to prevent erroneous updates being created against a linked template, I starting using a CI/CD process involving thorough testing and roll-back operations. This blog will go over some examples of how someone can setup some tests and rollback operations for deploying linked templates.

Before I start going through the details of the first test, I need to mention that the testing framework that I use for all of my tests is Pester. If you haven't used Pester before, you can have a look at the project [here](https://github.com/pester/Pester) to see how everything is setup. You are by no means required to use Pester when implementing this workflow yourself. I am comfortable with powershell, so this is the framework I choose to use.

Ok, so the first test that I run against my template leverages the `Test-AzureRmResourceGroupDeployment`. This command will perform a mock deployment POST request against my template and supplied parameters, and will spit back a response that I can parse and send to some pester tests to verify that I am getting back the results I intend. The HTTP response is only sent back in debug mode. To be able to grab the response data, we can use redirects after we change powershell's debug preference. The code for this looks like:
```
$DebugPreference = 'continue'
$armParams = @{
    ResourceGroupName = $resourceGroupName
    TemplateFile = $templateFilePath
    config = $config
}
$output = Test-AzureRmResourceGroupDeployment @armParams 5>&1
$DebugPreference = 'silentlycontinue'
```

The first line sets the debug preference to `continue` so we can get the debug output. The `$armParams` object works just like the `New-AzureRmResourceGroupDeployment` cmdlet. So we need to define what resource group we are deploying to, what template we are using, and any parameters that are needed in the template. Next, we can create an `$output` variable to capture the results. Notice that we are redirecting the debug output to stdout. For more information on redirects in powershell, Microsoft docs has a pretty thorough write up on this [here](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_redirection?view=powershell-6). After the command has run, we will want to change our debug preference back to `silenlycontinue`, or whatever the default preference is for your sessions.  

I won't post what the `$output` variable contains here, as it is very verbose, but what we are looking to capture from the output is the HTTP Response. Here is a snippet of what the looks:
```
==================================== HTTP RESPONSE ====================================
Status Code:
OK
```

Underneath this output contains the body of the response that we can parse for the values we need to test against. Here is the code that I use to get this data:
```
for ($i = 0; $i -lt $output.count; $i++) {
    if ($output[$i] -like "*HTTP RESPONSE*") {
        $lineNumber = $i;
        break
    }
}
$output = ($output[$lineNumber] -split "Body:")[1] | convertfrom-json
```

The `$output` variable now contains a powershell object with all the values returned from the mock arm template deploy. These values are located in the properties property of `$output`. Here is a sample of what this looks like:
```
templateHash       : 12243808956518658009
parameters         : @{config=}
mode               : Incremental
provisioningState  : Succeeded
timestamp          : 2018-02-26T07:57:16.698988Z
duration           : PT0S
correlationId      : 6e19c873-4539-410b-96a9-dfeae99d0b5c
providers          : {@{namespace=Microsoft.Web; resourceTypes=System.Object[]}, @{namespace=Microsoft.Insights; resourceTypes=System.Object[]}}
dependencies       : {@{dependsOn=System.Object[]; 
                     id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.Web/sites/apitest-dev; 
                     resourceType=Microsoft.Web/sites; resourceName=apitest-dev}, @{dependsOn=System.Object[]; id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.Web/sites/apitest-dev/slots/apitest-dev-staging-slot; 
                     resourceType=Microsoft.Web/sites/slots; resourceName=apitest-dev/apitest-dev-staging-slot}, @{dependsOn=System.Object[]; id=/subs
                     criptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.Web/sites/apitest-dev/slots/a
                     pitest-dev-staging-slot/extensions/MSDeploy; resourceType=Microsoft.Web/sites/slots/extensions; 
                     resourceName=apitest-dev/apitest-dev-staging-slot/MSDeploy}}
validatedResources : {@{apiVersion=2014-11-01; id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.W
                     eb/serverFarms/apitest-dev-plan; name=apitest-dev-plan; type=Microsoft.Web/serverFarms; location=westus; properties=}, 
                     @{apiVersion=2014-04-01; id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.In
                     sights/components/apitest-dev; name=apitest-dev; type=Microsoft.Insights/components; location=eastus; tags=; properties=}, 
                     @{dependsOn=System.Object[]; apiVersion=2015-04-01; 
                     id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.Web/sites/apitest-dev; 
                     name=apitest-dev; type=Microsoft.Web/sites; location=westus; identity=; properties=}, @{dependsOn=System.Object[]; 
                     apiVersion=2015-04-01; id=/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/yourResourceGroup/providers/Microsoft.Web/
                     sites/apitest-dev/slots/apitest-dev-staging-slot; name=apitest-dev/apitest-dev-staging-slot; 
                     type=Microsoft.Web/sites/slots; location=westus; properties=}...}
```

Next, I need to create an array of objects to pass to my pester tests. The first object is my `$config` object, which houses all of the parameters needed in my arm template deploy. The second object is the `$output` of the mock template deploy. The code is pretty straight-forward:

```
$result = @($config, $output)
```

Now that I have this `$result` object, I can start to parse out the values into separate variables for testing. The example I will use is in reference to testing a standardized web api template. In my pester script, I have created an array of resources I expect to be in my template:
```
$expectedResources = @(
    'Microsoft.Web/serverFarms',
    'Microsoft.Insights/components',
    'Microsoft.Web/sites',
    'Microsoft.Web/sites/slots',
    'Microsoft.Web/sites/slots/extensions'
)
```

I then start to split apart the `$result` object into testable sections:
```
$armConfig = $Result[0]
$armOutput = $Result[1].properties
$resources = $armOutput.validatedResources
$serverFarms = $resources | where type -eq $expectedResources[0]
$appInsights = $resources | where type -eq $expectedResources[1]
$webApiSite = $resources | where type -eq $expectedResources[2]
$webApiSlot = $resources | where type -eq $expectedResources[3]
$msDeploy = $resources | where type -eq $expectedResources[4]
```

I can then take each of the expected resources, and start to run tests against them. The first thing I am testing is the actual deployment. I am making sure that the deployment was successful, that the deployment was incremental, and that the deployment had the types of resources I expected. Here are what those tests look like along with the matching results:

```
Context 'Test-Deploy of API Template' {

    It 'Should be a successful deployment' {
        $armOutput.provisioningState | Should -Be 'Succeeded'
    }

    It 'Should have mode of Incremental' {
        $armOutput.mode | Should -Be 'Incremental'
    }

    It "Should have resources of $($expectedResources | sort )" {
        ($resources.type | sort) | Should -Be ($expectedResources | sort)
    }
}
```
```
  Describing Testing Deployment of the Standard Linked API Template

    Context Test-Deploy of API Template
      [+] Should be a successful deployment 773ms
      [+] Should have mode of Incremental 113ms
      [+] Should have resources of Microsoft.Insights/components Microsoft.Web/serverFarms Microsoft.Web/sites Microsoft.Web/sites/slots Microsoft.Web/sites/
slots/extensions 40ms
```

The subsequent test I perform relate to the values in each of the expected resources. Here is an example of the tests associated with the ServerFarms resource along with the matching results:
```
Context 'Configuration for ServerFarms resource' {
    
    It 'Should have api version of 2014-11-01' {
        $serverFarms.apiVersion | Should -Be '2014-11-01'
    }

    It "Should have name of $($armconfig.api.servicePlanName)" {
        $serverFarms.name | Should -Be $armconfig.api.servicePlanName
    }

    It "Should have location of $($armconfig.api.siteLocation)" {
        $serverFarms.location | Should -Be $armconfig.api.siteLocation
    }

    It "Should have property name of $($armconfig.api.servicePlanName)" {
        $serverFarms.properties.name | Should -Be $armconfig.api.servicePlanName
    }

    It "Should have property sku of $($armconfig.api.servicePlanSku)" {
        $serverFarms.properties.sku | Should -Be $armconfig.api.servicePlanSku
    }

    It "Should have property workerSize of $($armconfig.api.servicePlanInstances)" {
        $serverFarms.properties.workerSize | Should -Be $armconfig.api.servicePlanInstances
    }
}
```
```
    Context Configuration for ServerFarms resource
      [+] Should have api version of 2014-11-01 135ms
      [+] Should have name of apitest-dev-plan 51ms
      [+] Should have location of westus 32ms
      [+] Should have property name of apitest-dev-plan 32ms
      [+] Should have property sku of Standard 43ms
      [+] Should have property workerSize of 1 40ms
```

If all of my tests pass for the template, I then start the process of deploying my arm template to our artifact store and uploading it to an azure storage account for testing. It is important to note that I tag all of my templates with an auto-generated build number so that I can trace my artifact back to a specific build. Now that I have my template artifact for my web api, I can upload this to blob storage to test a deploy. The first thing that I am going to do is copy the current version of the template and append `.old` to the end of the filename (i.e. `webapi-template.json.old`). We do this so that when we upload the new version, we have a working old version that we know we can rollback to if deployment tests fail. We can upload the new version of the template to replace the current version. Here is a snippet of what that code might look like if you are using azure blob storage to house your arm templates:
```
if ($revert) {
    $srcBlobName = "$templateType-template.json.old"
    $destBlobName = "$templateType-template.json"
} else {
    $srcBlobName = "$templateType-template.json"
    $destBlobName = "$templateType-template.json.old"
}

$blobParams = @{
    Container = 'templates'
    Context = $storageInfo.Context
    Blob = $srcBlobName
}
$srcBlob = Get-AzureStorageBlob @blobParams -erroraction silentlycontinue

if ($srcBlob) {
    #Renaming old template
    $copyParams = @{
        SrcBlob = $srcBlobName
        SrcContainer = 'arm-templates'
        DestContainer = 'arm-templates'
        DestBlob = $destBlobName
        Context = $storageInfo.Context
        Force = $True
    }
    $destBlob = Start-AzureStorageBlobCopy @copyParams
    #Copy over Metadata for version
    $destBlockBlob = [Microsoft.WindowsAzure.Storage.Blob.CloudBlockBlob] $destBlob.ICloudBlob
    $destBlockBlob.Metadata['version'] = $srcBlob.ICloudBlob.Metadata['version']
    $destBlockBlob.SetMetadata()
}

# Adding new template
if (-not $Revert) {
    $uploadParams = @{
        Container = $containerName
        File = $TemplateFilePath
        Blob = $srcBlobName
        Context = $StorageInfo.Context
        Metadata = @{'version' = $BuildNumber}
        Force = $true
        Confirm = $false
    }
    $result = Set-AzureStorageBlobContent @uploadParams
}
```

To walk through this code a bit, first we are setting the source and destination name. The source and destination flip based on whether or not we are deploying or rolling back the template, which is the purpose of the `$revert` switch. Next we test to see if a current version of the template exists. If it does, we copy the template and rename it using the `.old`. We also copy over any metadata included in the blob. I like to include the build version of the template in the metadata, which is why I copy it over. The last part is to deploy the new template to blob storage.

Now that we have the template deployed, we can run a test deployment using this template. So instead of running `Test-AzureRmResourceGroupDeployment`, this time we will run `New-AzureRmResourceGroupDeployment`.
```
$result += New-AzureRmResourceGroupDeployment @armParams
```

It's not shown here, but in the beginning of my deployment script, I am declaring an empty array called `$result`. Anytime I run an action that I can test, I create a new item in the array with the results of the action I ran. At the end of my deployment script, I then send the `$result` object to be tested, just like we saw in the above example. Here is what my testable result variables look like:
```
# Grab all resources in resource group and create testable sections
$loginResults = $result[0]
$newResourceGroupResults = $result[1]
$stopSlotResults = $result[2]
$deployResults = $result[3]
$appsettingResults = $result[4]
$startSlotResults = $result[5]
$armConfig = $result[6]
```

Below, I am showing the tests and results associated with the `$deployResults`, which contains the results of `New-AzureRmResourceGroupDeployment @armParams`.

```
Context 'Deploying ARM template' {
    
    It "Should have deployment name of $($armConfig.buildNumber)" {
        $deployResults.deploymentName | Should -Be $armConfig.buildNumber
    }

    It 'Should have mode of Incremental' {
        $deployResults.mode | Should -Be 'Incremental'
    }
    
    It 'Should have one parameter' {
        $deployResults.parameters.count | Should -Be 1
    }

    It 'Should have one parameter name of config' {
        $deployResults.parameters.keys | Should -Be 'config'
    }

    It 'Should have one parameter type of SecureObject' {
        $deployResults.parameters['config'].type | Should -Be 'SecureObject'
    }
}
```

```
    Context Deploying ARM template
      [+] Should have deployment name of 1001 106ms
      [+] Should have mode of Incremental 17ms
      [+] Should have one parameter 18ms
      [+] Should have one parameter name of config 22ms
      [+] Should have one parameter type of SecureObject 12ms
```

I have this setup so that if the deploy tests pass in our dev subscription, the same process occurs in our prod subscription. If one of the deploy tests fails, then the arm template rolls back to the previous version using the `$revert` switch shown in the above code that handles deploying the new template to blob storage.