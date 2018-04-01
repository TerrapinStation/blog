# Creating a helper function to allow non-interactive login to Azure AD via a service principal

Testing is a staple for teams that have CI/CD workflows. One use case our team utilizes testing for is when deploying web APIs to Azure. Our current process uses deployment slots and we test that the slot deployment was successful. If that test passes, we swap the slots and test the live slot. If the live slot tests succeed, we deploy the api to our API Gateway for consumption.

This whole process was working great until we changed the requirements to protect **all** APIs with Azure AD. All of our tests, as well as any API gateway deployments were failing due to the fact that our service principal that handled all the automation behind our deployments was getting 401 errors since the APIs were now requiring the user to login to Azure AD before being able to access the API.

This post will go into detail on how we were able to generate authentication tokens to an Azure resource using our service principal, so that all of our automation would continue to work as expected.

Here are the pre-requisites to be able to perform the solution presented below:

- Microsoft.IdentityModel.Clients.ActiveDirectory.dll. I'm using version 3.19.2 [Link](https://www.nuget.org/packages/Microsoft.IdentityModel.Clients.ActiveDirectory/3.19.2)

- AzureAD module. Im using version 2.0.0.131 [Link](https://www.powershellgallery.com/packages/AzureAD/2.0.0.131)

- A service principal with access to sign in to Azure AD and read directory data.

- The aforementioned service principal should be enabled to login to Azure via certificate.

The method we took to solving this was to create a private helper function in our deployment powershell module to be used to grab a token based on a given resource. There are many different resources that you can get a token for, and the expected syntax that is needed to grab a resource differs between resources. The resource that we will be referencing in this post is an Azure app registration that is tied to an Azure web app. In order to get a token for our app registration, we need to provide the application ID of the app registration. We handle this by taking in a Application Name parameter and querying Azure AD for the app ID. In order to query Azure AD, we need to first connect to Azure AD using our service principal via it's associated certificate. Here is what that might look like:
```
$connectParams = @{
      TenantId = $tenantID
      ApplicationId = $servicePrincipalID
      CertificateThumbprint = $servicePrincipalThumbprint
    }
    Connect-AzureAD @connectParams | Out-Null
```
We have a way to generate our `$tenantID`, `$servicePrincipalID` and `$servicePrincipalThumbprint` using some config data, so you will want to make sure that you also have a way to generate those variables specific to your azure subscription in order for the rest of the script to work correctly.

The next step queries Azure AD for the given application name:
```
$app = Get-AzureAdApplication -filter "displayname eq '$ApplicationName'"
if (-not $app) {
    throw "$ApplicationName doesn't exist as an Azure AD Application."
} else {
$resource = $app.AppId
}
```
If the application doesn't exist, we throw an error. If it does exist, we grab the application ID via `$app.AppId` and assign it to `$resource`.

The next step requires the Microsoft.IdentityModel.Clients.ActiveDirectory dll. So if you haven't already, this needs to be added as a type in the script so that we can leverage the .NET methods required for generating our tokens. We installed the nuget package into our C:\tools directory, but you will want to point to wherever you installed the package. Here is how to load the DLL into the script:
```
Add-Type -Path "C:\tools\Microsoft.IdentityModel.Clients.ActiveDirectory.3.19.2\lib\net45\Microsoft.IdentityModel.Clients.ActiveDirectory.dll" -ErrorAction Stop
```

Once the DLL is loaded we can create our authentication context by pointing to our $tenantID. This can be done via:
```
$authContext = [Microsoft.IdentityModel.Clients.ActiveDirectory.AuthenticationContext]::new("https://login.windows.net/$tenantID)")
```

Next we will need to grab the service principal's certificate. This looks like:
```
$clientCert = Get-Item -Path "Cert:\LocalMachine\My\$servicePrincipalThumbPrint"
```
The location of the cert might vary based on where you installed it.

Next, we can create a credential object used to grab our token by passing in our service principals ID along with the `$clientCert` we just initialized.
```
$certCred = [Microsoft.IdentityModel.Clients.ActiveDirectory.ClientAssertionCertificate]::new($servicePrincipalID, $clientCert)
```

Then we can call the .NET method to acquire the token by passing in the `$resource` (which is the application ID) and our `$certCred` object we just created.
```
$authResult = $authContext.AcquireTokenAsync($resource,$certCred)
```

Lastly, we can create a header for the token that we use to pass to our REST calls that we need to make for our testing and API gateway deployments.
```
$Token = $authResult.Result.CreateAuthorizationHeader()
```

This token will look something like this:
```
Bearer eyJ0eAXiOiJKV1QiLCJhbGciOiJSUzI1NiIsIn1gdCI6IkZTaW11RnJGTm9DMNKHWEdtEdEzbk5aY2VE...
```
The `...` is representing a continuation of a much longer string than what I showed above.

So now that we have our token, we can start to pass this into our web requests via a header that will allow the service principal to non-interactively authenticate to Azure AD for the specific resource we generated the token for. This might look something like this:
```
$Request = @{
    URI = 'atestapi.azurewebsites.net'
    Method = 'GET'
    Headers = @{
        'Authorization' = $Token
        'Content-Type' = 'application/json'
    }
    UseBasicParsing = $true
}
Invoke-RestMethod @Request
```

Anytime we need to access a Azure AD protected API, we can grab an access token using the helper function described above that allows us to non-interactively login to Azure AD so that we can continue to leverage automated testing and API deployments through our CI/CD workflow.