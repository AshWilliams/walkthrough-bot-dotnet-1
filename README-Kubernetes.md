## Creating the Kubernetes cluster and connecting the bot to the cluster

#### Installing Azure CLI

Azure CLI will be the hand tool for almost everything in this laboratory, let's download and configure it following the instructions here: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest.

Note: In some scenarios we can use the Bash terminal hosted in the Azure Portal, however there are some configurations for Kubernetes that requires a local machine configured with Azure CLI, that's why I decided to installed and configured from the beginning.

#### Installing Docker CE

Go to the Docker portal https://www.docker.com/get-started and install Docker in your local machine.

Docker will be used to build the images and push them to the ACR via CLI in the local machine, this is the preamble before start working with Kubernetes.

#### Azure Container Registry Configuration

Once we have configured the Azure CLI, its time to open a terminal.

Let's login to Azure.

```bash
az login
```

Display the accounts associated to your subscription.

```bash
az account show
```

Select the account you want to use.

```bash
az account set --subscription 
```

Let's create the resource group that we are going to use to allocate the ACR.

```bash
az group create --name wbd-acr --location westus
```

Let's create the Azure Container Registry (ACR).

```bash
az acr create --resource-group wbd-acr --name wbdacr --sku Basic --admin-enabled true
```

Get the credentials to access to the ACR.

```bash
az acr credential show --resource-group wbd-acr --name wbdacr --output yaml
```

You should get something like this:

```yaml
- name: password
  value: REAL_PASSWORD <-REAL PASSWORD SHOULD APPEAR HERE.
- name: password2
  value: REAL_PASSWORD <-REAL PASSWORD SHOULD APPEAR HERE.
username: wbdacr
```

Take note of the following, you will use information later.

```bash
Username: wbdacr
Password: REAL_PASSWORD
Login Server: wbdacr.azurecr.io
```

#### Building Docker in Bot Project

1. Ensure you have Docker running in your local machine, you have the latest version of the GitHub repo and you have successfully accomplished the [bot  programming laboratory](README-BotBuilderV4.md).

2. Let's backup the appsettings.json content in a notepad and then we are going to delete any settings. We are going to use the settings later.

From this:

```json
{
    "MicrosoftAppId": "MICROSOFT_BOT_APP_ID",
    "MicrosoftAppPassword": "MICROSOFT_BOT_APP_PASSWORD",
    "TranslatorTextAPIKey": "AZURE_TRANSLATOR_KEY",
    "BotVersion": "BOT_VERSION",
    "LuisName01": "LUIS_NAME (e.g. Reminders)",
    "LuisAppId01": "LUIS_APPLICATION_ID",
    "LuisAuthoringKey01": "LUIS_AUTHORING_KEY",
    "LuisEndpoint01": "LUIS_ENDPOINT"
}
```

To this:

```json
{
}
```

<b>Why we delete whole settings?</b> We don't want to encapsulate the credentials in the container image, the image MUST be agnostic to every environment.

3. Open a terminal (as Administrator) pointing the folder `source\1. start\WBD\` folder and run the following command:

```json
docker image build . --tag "latest"
```

Wait until nuget packages are being reinstalled and the project finishes the build process (the project should be successfully compiled but is not ready to be executed).

You should be able to see something like this:

```bash
Sending build context to Docker daemon  2.576MB
Step 1/15 : FROM microsoft/dotnet:2.1.5-aspnetcore-runtime AS base
 ---> 84e946977e6f
Step 2/15 : WORKDIR /app
 ---> Using cache
 ---> 9581dfa8f515
Step 3/15 : EXPOSE 80
 ---> Using cache
 ---> f777b9570039
Step 4/15 : FROM microsoft/dotnet:2.1.403-sdk AS build
2.1.403-sdk: Pulling from microsoft/dotnet
bc9ab73e5b14: Pull complete
193a6306c92a: Pull complete
e5c3f8c317dc: Pull complete
a587a86c9dcb: Pull complete
290de22a9506: Pull complete
f378ac4c5856: Pull complete
795266ce0809: Pull complete
Digest: sha256:8e9138b0aebf298b78d299e09a32809fbe492e4a8d8cecf87f839b298309b2ed
Status: Downloaded newer image for microsoft/dotnet:2.1.403-sdk
 ---> 6baac5bd0ea2
Step 5/15 : WORKDIR /src
 ---> Running in 0f44907054af
Removing intermediate container 0f44907054af
 ---> 59b3b40b383d
Step 6/15 : COPY WBD.csproj ./
 ---> 62779ed59220
Step 7/15 : RUN dotnet restore ./WBD.csproj
 ---> Running in 4d72e6cc74a9
  Restoring packages for /src/WBD.csproj...
  Installing System.Security.Principal.Windows 4.0.0.
  Installing System.Threading.Overlapped 4.0.1.
  Installing System.Threading.Overlapped 4.3.0.
  Installing System.Text.Encodings.Web 4.4.0.
  Installing Microsoft.AspNetCore.Http.Features 2.0.1.
  Installing System.Runtime.CompilerServices.Unsafe 4.4.0.
  Installing Microsoft.IdentityModel.Tokens 5.2.1.
  Installing Microsoft.IdentityModel.Logging 5.2.1.
  Installing System.Net.NetworkInformation 4.1.0.
  Installing System.Net.Requests 4.0.11.
  Installing System.Net.WebHeaderCollection 4.0.1.
  Installing System.Net.NetworkInformation 4.3.0.
  Installing System.Security.Principal 4.0.1.
  Installing System.Security.Claims 4.0.1.
  Installing runtime.native.System.Net.Security 4.0.1.
  Installing Microsoft.Recognizers.Text.NumberWithUnit 1.1.0.
  Installing Microsoft.Recognizers.Text 1.1.0.
  Installing Microsoft.VisualStudio.Azure.Containers.Tools.Targets 1.1.1.
  Installing Microsoft.Bot.Builder 4.0.8.
  Installing Microsoft.Bot.Builder.AI.QnA 4.0.8.
  Installing System.ValueTuple 4.4.0.
  Installing MongoDB.Driver 2.7.0.
  Installing StackExchange.Redis.StrongName 1.2.6.
  Installing Microsoft.Bot.Schema 4.0.8.
  Installing Microsoft.Bot.Builder.AI.Luis 4.0.8.
  Installing Microsoft.Bot.Connector 4.0.8.
  Installing MongoDB.Bson 2.7.0.
  Installing System.Diagnostics.DiagnosticSource 4.5.1.
  Installing Microsoft.Bot.Builder.Integration.AspNet.Core 4.0.8.
  Installing System.Security.Principal.Windows 4.5.1.
  Installing Microsoft.AspNetCore.All 2.1.5.
  Installing Microsoft.Recognizers.Text.Number 1.1.1.
  Installing Microsoft.Bot.Builder.Dialogs 4.0.8.
  Installing System.Runtime.CompilerServices.Unsafe 4.5.2.
  Installing MongoDB.Driver.Core 2.7.0.
  Installing Microsoft.AspNetCore.Http.Abstractions 2.0.1.
  Installing Microsoft.AspNetCore.Hosting.Server.Abstractions 2.0.1.
  Installing Microsoft.Extensions.FileProviders.Abstractions 2.0.0.
  Installing Microsoft.Extensions.DependencyInjection.Abstractions 2.0.0.
  Installing Microsoft.Bot.Builder.Azure 4.0.8.
  Installing Microsoft.Extensions.Hosting.Abstractions 2.0.1.
  Installing Cognitive.Face.Dotnet 1.0.2.
  Installing Microsoft.Extensions.Logging.Abstractions 2.0.0.
  Installing SourceLink.Create.CommandLine 2.8.1.
  Installing System.ComponentModel.TypeConverter 4.1.0.
  Installing System.Buffers 4.4.0.
  Installing Microsoft.Extensions.Primitives 2.0.0.
  Installing Microsoft.Win32.Registry 4.0.0.
  Installing Newtonsoft.Json 10.0.3.
  Installing System.Threading.ThreadPool 4.0.10.
  Installing System.Threading.Thread 4.0.0.
  Installing System.IdentityModel.Tokens.Jwt 5.2.1.
  Installing Microsoft.IdentityModel.Protocols 5.2.1.
  Installing Microsoft.Rest.ClientRuntime.Azure 3.3.13.
  Installing System.ComponentModel 4.0.1.
  Installing Microsoft.Azure.CognitiveServices.Language.LUIS.Runtime 2.0.0.
  Installing System.ComponentModel.Primitives 4.1.0.
  Installing Microsoft.Rest.ClientRuntime 2.3.13.
  Installing Microsoft.IdentityModel.Protocols.OpenIdConnect 5.2.1.
  Installing WindowsAzure.Storage 9.3.1.
  Installing System.Collections.NonGeneric 4.0.1.
  Installing System.Diagnostics.Process 4.1.0.
  Installing Microsoft.Extensions.Configuration 2.0.1.
  Installing Microsoft.Net.Http.Headers 2.0.1.
  Installing Microsoft.AspNetCore.Hosting.Abstractions 2.0.1.
  Installing Microsoft.AspNetCore 2.1.4.
  Installing Microsoft.AspNetCore.Authentication 2.1.2.
  Installing Microsoft.AspNetCore.Authentication.MicrosoftAccount 2.1.2.
  Installing Microsoft.AspNetCore.Identity.EntityFrameworkCore 2.1.3.
  Installing Microsoft.AspNetCore.Mvc 2.1.3.
  Installing Microsoft.AspNetCore.Identity 2.1.3.
  Installing Microsoft.Extensions.Identity.Stores 2.1.3.
  Installing Microsoft.AspNetCore.Authentication.Cookies 2.1.2.
  Installing Microsoft.Extensions.Caching.SqlServer 2.1.2.
  Installing Microsoft.AspNetCore.Authentication.Facebook 2.1.2.
  Installing Microsoft.Extensions.Identity.Core 2.1.3.
  Installing Microsoft.AspNetCore.Authentication.JwtBearer 2.1.2.
  Installing Microsoft.AspNetCore.CookiePolicy 2.1.2.
  Installing Microsoft.AspNetCore.Authentication.OAuth 2.1.2.
  Installing Microsoft.AspNetCore.Authorization.Policy 2.1.2.
  Installing Microsoft.AspNetCore.Mvc.Cors 2.1.3.
  Installing Microsoft.AspNetCore.Authentication.Twitter 2.1.2.
  Installing Microsoft.AspNetCore.Authentication.WsFederation 2.1.2.
  Installing Microsoft.AspNetCore.Mvc.Abstractions 2.1.3.
  Installing Microsoft.AspNetCore.Mvc.ApiExplorer 2.1.3.
  Installing Microsoft.AspNetCore.Http.Connections 1.0.4.
  Installing Microsoft.AspNetCore.Mvc.Analyzers 2.1.3.
  Installing Microsoft.AspNetCore.Authentication.Google 2.1.2.
  Installing Microsoft.AspNetCore.Mvc.Localization 2.1.3.
  Installing Microsoft.AspNetCore.Mvc.TagHelpers 2.1.3.
  Installing Microsoft.AspNetCore.Mvc.DataAnnotations 2.1.3.
  Installing Microsoft.AspNetCore.Http.Connections.Common 1.0.4.
  Installing Microsoft.AspNetCore.Mvc.Formatters.Xml 2.1.3.
  Installing Microsoft.AspNetCore.Mvc.RazorPages 2.1.3.
  Installing Microsoft.AspNetCore.Connections.Abstractions 2.1.3.
  Installing Microsoft.AspNetCore.Mvc.Razor 2.1.3.
  Installing Microsoft.AspNetCore.Razor.Runtime 2.1.2.
  Installing Microsoft.AspNetCore.Razor 2.1.2.
  Installing Microsoft.AspNetCore.Mvc.Formatters.Json 2.1.3.
  Installing System.IO.Pipelines 4.5.2.
  Installing Microsoft.AspNetCore.Authorization 2.1.2.
  Installing Microsoft.EntityFrameworkCore.Abstractions 2.1.4.
  Installing Microsoft.AspNetCore.Mvc.Razor.Extensions 2.1.2.
  Installing WindowsAzure.Storage 9.3.0.
  Installing Microsoft.AspNetCore.Razor.Language 2.1.2.
  Installing Microsoft.AspNetCore.Mvc.ViewFeatures 2.1.3.
  Installing Microsoft.EntityFrameworkCore.InMemory 2.1.4.
  Installing Microsoft.Extensions.Caching.Memory 2.1.2.
  Installing Microsoft.AspNetCore.SignalR.Protocols.Json 1.0.4.
  Installing Microsoft.EntityFrameworkCore.Sqlite.Core 2.1.4.
  Installing Microsoft.AspNetCore.SignalR 1.0.4.
  Installing Microsoft.AspNetCore.Server.Kestrel.Transport.Abstractions 2.1.3.
  Installing Microsoft.AspNetCore.Identity.UI 2.1.3.
  Installing Microsoft.AspNetCore.SignalR.Common 1.0.4.
  Installing Microsoft.CodeAnalysis.Razor 2.1.2.
  Installing Microsoft.AspNetCore.Server.Kestrel.Transport.Libuv 2.1.3.
  Installing Microsoft.EntityFrameworkCore.SqlServer 2.1.4.
  Installing Microsoft.Extensions.Caching.Abstractions 2.1.2.
  Installing Microsoft.AspNetCore.Server.Kestrel 2.1.3.
  Installing Microsoft.AspNetCore.Server.Kestrel.Transport.Sockets 2.1.3.
  Installing Microsoft.EntityFrameworkCore.Sqlite 2.1.4.
  Installing Microsoft.EntityFrameworkCore.Design 2.1.4.
  Installing Microsoft.EntityFrameworkCore.Relational 2.1.4.
  Installing Microsoft.AspNetCore.Authentication.OpenIdConnect 2.1.2.
  Installing Microsoft.AspNetCore.Server.IISIntegration 2.1.2.
  Installing Microsoft.AspNetCore.Server.Kestrel.Https 2.1.3.
  Installing Microsoft.EntityFrameworkCore.Tools 2.1.4.
  Installing Microsoft.AspNetCore.Server.Kestrel.Core 2.1.3.
  Installing Microsoft.EntityFrameworkCore.Analyzers 2.1.4.
  Installing Microsoft.AspNetCore.SignalR.Core 1.0.4.
  Installing Microsoft.AspNetCore.Mvc.Core 2.1.3.
  Installing Microsoft.Azure.DocumentDB.Core 1.9.1.
  Installing Microsoft.Extensions.Caching.Redis 2.1.2.
  Installing Microsoft.AspNetCore.SignalR.Redis 1.0.4.
  Installing Microsoft.EntityFrameworkCore 2.1.4.
  Installing System.Collections.Immutable 1.4.0.
  Installing Microsoft.Recognizers.Text 1.1.1.
  Installing Microsoft.Recognizers.Text.Choice 1.1.0.
  Installing Microsoft.Recognizers.Text.DateTime 1.1.0.
  Installing System.Threading.Tasks.Extensions 4.4.0.
  Installing DnsClient 1.0.7.
  Installing System.Net.Security 4.0.0.
  Installing System.Collections.Specialized 4.0.1.
  Installing System.Security.SecureString 4.0.0.
  Installing System.Diagnostics.TraceSource 4.0.0.
  Generating MSBuild file /src/obj/WBD.csproj.nuget.g.props.
  Generating MSBuild file /src/obj/WBD.csproj.nuget.g.targets.
  Restore completed in 9.81 sec for /src/WBD.csproj.
Removing intermediate container 4d72e6cc74a9
 ---> 64ef96f12f87
Step 8/15 : COPY . .
 ---> fa3a739c314b
Step 9/15 : RUN dotnet build WBD.csproj -c Release -o /app
 ---> Running in 2ba4a47fead6
Microsoft (R) Build Engine version 15.8.169+g1ccb72aefa for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restoring packages for /src/WBD.csproj...
  Generating MSBuild file /src/obj/WBD.csproj.nuget.g.props.
  Generating MSBuild file /src/obj/WBD.csproj.nuget.g.targets.
  Restore completed in 971.77 ms for /src/WBD.csproj.
  WBD -> /app/WBD.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:05.37
Removing intermediate container 2ba4a47fead6
 ---> 8ad579d146b0
Step 10/15 : FROM build AS publish
 ---> 8ad579d146b0
Step 11/15 : RUN dotnet publish WBD.csproj -c Release -o /app
 ---> Running in f6b80d30bbf4
Microsoft (R) Build Engine version 15.8.169+g1ccb72aefa for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 54.84 ms for /src/WBD.csproj.
  WBD -> /src/bin/Release/netcoreapp2.1/WBD.dll
  WBD -> /app/
Removing intermediate container f6b80d30bbf4
 ---> 4dd4fd6f3afc
Step 12/15 : FROM base AS final
 ---> f777b9570039
Step 13/15 : WORKDIR /app
 ---> Running in 90d7b5358971
Removing intermediate container 90d7b5358971
 ---> 402205bc3e14
Step 14/15 : COPY --from=publish /app .
 ---> 1c3d427c6fdb
Step 15/15 : ENTRYPOINT ["dotnet", "WBD.dll"]
 ---> Running in 37f981f3ccfb
Removing intermediate container 37f981f3ccfb
 ---> 8daa1a9317c3
Successfully built 8daa1a9317c3
Successfully tagged latest:latest
SECURITY WARNING: You are building a Docker image from Windows against a non-Windows Docker host. All files and directories added to build context will have '-rwxr-xr-x' permissions. It is recommended to double check and reset permissions for sensitive files and directories.
```

4. Let's login to ACR.

```bash
docker login wbdacr.azurecr.io --username wbdacr --password REAL_PASSWORD
```

5. Create an alias of the image.

Note: If you notice at the end of the build log we have a `Successfully built 8daa1a9317c3` message, that identificator means the IMAGE ID of the application.

Let's apply an alias in the ACR.

```bash
docker tag 8daa1a9317c3 wbdacr.azurecr.io/wbd
```

6. Push the image to the ACR.

```bash
docker push wbdacr.azurecr.io/wbd
```

You should be able to see something like this:

```bash
The push refers to repository [wbdacr.azurecr.io/wbd]
32036fcc5590: Pushed
3aa5075795e2: Pushed
5d579c33053b: Pushed
8b1e393851d9: Pushed
799e74090312: Pushed
237472299760: Pushed
latest: digest: sha256:eda6bb817045e00fa9a879eae328d151c7f25e69829fa400b968289c9bb52747 size: 1581
```

7. Congratulations, if you go to your Azure Portal, in your ACR repositories, you will be able to see the image pushed.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-acr.png" />
</div>

