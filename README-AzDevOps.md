## Intro

Welcome to the walkthrough-bot-dotnet.

## Latest Builds

N/A.

## Prerequisite to run this bot locally

#### Software Prerequisites

1. Active Azure subscription
2. VS Code o Visual Studio 2017 Community
3. Azure DevOps free account (https://dev.azure.com/)
4. .Net Core Installed (https://www.microsoft.com/net/download)
5. Git for Windows, Linux or MacOS are optional (https://git-scm.com/downloads)
6. Bot Framework V4 Emulator (https://github.com/Microsoft/BotFramework-Emulator/releases/tag/v4.1.0)

## Build the Bot (Laboratory)

#### Clone the project from GitHub repo

`git clone https://github.com/rcervantes/walkthrough-bot-dotnet.git`

#### Using Visual Studio 2017 (Any Version)

1. Based on the webinar explanation go to `source\1. start\` folder and open the WBD.sln file in Visual Studio 2017.

2. Wait until nuget packages are being reinstalled.

3. Build the WBD project (the project should be successfully compiled but is not ready to be executed).

#### Using Visual Studio Code (Any Version)

1. Open Visual Studio Code and open the following folder `source\1. start\WBD\`.

2. Open a terminal pointing the folder `source\1. start\WBD\` folder and run the following command: <b>dotnet build</b>, wait until nuget packages are being reinstalled and the project finishes the build process (the project should be successfully compiled but is not ready to be executed).

#### Translator Text Configuration

Go to https://portal.azure.com/ and get successfully sign-in with your Employee or Microsoft account.

Steps:

1. Create a Translator Text resource with the following configuration:

    - Name: translator(uniqueid) e.g. translator017.
    - Subscription: your subscription.
    - Pricing tier: S1 (Pay as you go).
    - Resource Group: Create new with the same resource name, e.g. translator017.

2. Once the resource has been deployed go to the resource and click in <b>Resource Management->Keys</b> and take note of the `Key 1` in a notepad (we are going to use this information later to configure the bot).

#### LUIS Configuration

1. Go to https://www.luis.ai/ and get successfully sign-in with your Employee or Microsoft account.

2. Once you are signed-in, go to My Apps and click `Import new app`, select the file: Reminders.json located in the folder `source\3. models\` and finally click done.

3. A new LUIS application has been created (this application contains three intents: Calendar.Add,Calendar.Find and None, each intent has been filled with a bunch of utterances as an example).

4. Go to Reminders app and click on Train button, once you have trained the model click Publish to make public the API service.

5. Once the service has been trained and published go to manage and take note of the `Application ID (from Application Information tab)`, `Authoring Key (from Keys and Endpoints tab)`, `Endpoint (from Keys and Endpoints tab)` in a notepad (we are going to use this information later to configure the bot).

6. Everytime you add, remove or update utterances a new train and publish process is required to train and expose the latest version.

#### Bot Configuration

Return to Visual Studio 2017 or Visual Studio Code open and update the `appsettings.json` file in the root of the bot project.

Your appsettings.json file should look like this
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

For local development and debugging MicrosoftAppId and MicrosoftAppPassword can be empty, these settings are used in Azure deployment.

#### Getting Dirty (Let's Code!)

1. In the Startup.cs file replace line 39 with:

```csharp
Settings.MicrosoftAppId = Configuration.GetSection("MicrosoftAppId")?.Value;
Settings.MicrosoftAppPassword = Configuration.GetSection("MicrosoftAppPassword")?.Value;
Settings.TranslatorTextAPIKey = Configuration.GetSection("TranslatorTextAPIKey")?.Value;
Settings.BotVersion = Configuration.GetSection("BotVersion")?.Value;
Settings.LuisAppId01 = Configuration.GetSection("LuisAppId01")?.Value;
Settings.LuisName01 = Configuration.GetSection("LuisName01")?.Value;
Settings.LuisAuthoringKey01 = Configuration.GetSection("LuisAuthoringKey01")?.Value;
Settings.LuisEndpoint01 = Configuration.GetSection("LuisEndpoint01")?.Value;
```

<b>Note:</b> This code provides the correct settings from the `appsettings.json` file.

2. In the Startup.cs file replace line 77 with:

```csharp
var luisServices = new Dictionary<string, LuisRecognizer>();
var app = new LuisApplication(Settings.LuisAppId01, Settings.LuisAuthoringKey01, Settings.LuisEndpoint01);
var recognizer = new LuisRecognizer(app);
luisServices.Add(Settings.LuisName01, recognizer);
```

3. In the Startup.cs file replace line 81 with:

```csharp
var accessors = new BotAccessors(conversationState, userState, luisServices)
```

<b>Note:</b> These changes provides the correct configuration required to consume LUIS service from the bot and configure the accessor object.

4. In the Startup.cs file replace line 83 with:

```csharp
ConversationDialogState = conversationState.CreateProperty<DialogState>("DialogState"),
LanguagePreference = userState.CreateProperty<string>("LanguagePreference"),
IsReadyForLUISPreference = userState.CreateProperty<bool>("IsReadyForLUISPreference")
```

<b>Note:</b> This code provides the correct configuration for the accessors, LanguagePreference and IsReadyForLUISPreference will be use as part of the userState object.

5. In the Bot.cs file replace line 39 with:

```csharp
await dialogContext.BeginDialogAsync("MainDialog", null, cancellationToken);
```

<b>Note:</b> This code provides the initialization for the first dialog in the bot, everything starts from here.

6. In the Bot.cs file replace line 49 with:

```csharp
await accessors.IsReadyForLUISPreference.DeleteAsync(turnContext);
await accessors.LanguagePreference.DeleteAsync(turnContext);
await dialogContext.EndDialogAsync();
await dialogContext.BeginDialogAsync("MainDialog", null, cancellationToken);
```

<b>Note:</b> This code provides the end and begin of the dialog, deleting all values from preferences.

7. In the Bot.cs file replace line 63 with:

```csharp
string userLanguage = await accessors.LanguagePreference.GetAsync(turnContext, () => { return string.Empty; });
turnContext.Activity.Text = await TranslatorHelper.TranslateSentenceAsync(turnContext.Activity.Text, userLanguage, "en");
await turnContext.SendActivityAsync($"Sending to LUIS -> {turnContext.Activity.Text}");

// LUIS
var recognizerResult = await accessors.LuisServices[Settings.LuisName01].RecognizeAsync(turnContext, cancellationToken);
var topIntent = recognizerResult?.GetTopScoringIntent();
if (topIntent != null && topIntent.HasValue && topIntent.Value.intent != "None")
{
    await ProcessIntentAsync(turnContext, topIntent.Value.intent, topIntent.Value.score, cancellationToken);
}
else
{
    var response = @"No LUIS intents were found.";
    var message = await TranslatorHelper.TranslateSentenceAsync(response, "en", userLanguage);
    await turnContext.SendActivityAsync(message);
}
```

<b>Note:</b> This code provides the configuration required to call LUIS service using the Bot Builder SDK identify the correct top intent and the top score related with the utterance sent.

8. In the Bot.cs file replace line 83 with:

```csharp
animationCard = new AnimationCard
{
Title = $"Intent: {intent}",
Subtitle = $"Score: {score}",
Media = new List<MediaUrl>
{
    new MediaUrl()
    {
        Url = "https://i.gifer.com/HwZb.gif",
    },
},
};

reply.Attachments.Add(animationCard.ToAttachment());
```

<b>Note:</b> This code provides the animation card for Calendar_Add intent.

9. In the dialogs\LanguageDialog.cs replace line 20 with:

```csharp
RequestPhraseDialog,
ResponsePhraseDialog,
EndLanguageDialog
```

<b>Note:</b> This code provides the waterfall sequence that will be executed by the LanguageDialog.

10. In the dialogs\LanguageDialog.cs replace line 28 with:

```csharp
if (promptContext.Recognized.Value == null)
{
    await promptContext.Context.SendActivityAsync($"Sorry, but I'm expecting an string, send me another phrase");
}
else
{
    var value = promptContext.Recognized.Value;
    if (value.Length < 4)
    {
        await promptContext.Context.SendActivityAsync("Your phrase must be at least 4 characters long");
    }
    else
    {
        return true;
    }
}
```

<b>Note:</b> This code provides the validations required for the prompt used in RequestPhraseDialog.

11. In the dialogs\LanguageDialog.cs replace line 59 with:

```csharp
await accessors.LanguagePreference.SetAsync(step.Context, step.ActiveDialog.State["language"].ToString());
await accessors.IsReadyForLUISPreference.SetAsync(step.Context, true);
await accessors.UserState.SaveChangesAsync(step.Context, false, cancellationToken);
```

<b>Note:</b> This code provides the routine to save the preferences before end the LanguageDialog and return to MainDialog.

#### Running the Bot

Congratulations! if you are here is because your bot is almost done, you only need to verify the configuration of the ports, run the bot app and open the WBD.bot file from the emulator.

Before continue please open the WBD.csproj properties and validate the correct port: 

- Project Properties -> Debug -> Web Server Settings -> App Url: `http://localhost:3978/`.

- Now, open the file WBD.bot located in the root of the project and verify the line 9: `"endpoint": "http://localhost:3978/api/messages"`.

<b>Note:</b> Both configuration are pointing to localhost and the same port 3978 as default, you can set a new port in both places in case you need it.

Run your bot app and wait the web application be launched then open the Bot Framework Emulator (V4 Preview) and open the WBD.bot file you previously configured.

In the Bot Framework Emulator (V4 Preview) you will see an Endpoint called: Local, click it and your bot should be starting with the initial phrase configured in your dialog.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-emulator.png" />
</div>

#### Troubleshooting

If you have issues with your emulator, verify you have unchecked the option: Bypass ngrok for local address located in emulator settings. 

#### Completed Code

In case you want to review your code with the complete solution you can follow the previous configuration steps using the complete solution located in: `source\2. completed\WBD\`.

## Build Bot CI/CD Pipelines (Laboratory)

#### Creating Azure DevOps Account and Project

1. Go to https://dev.azure.com and get successfully sign-in with your Employee or Microsoft account.

2. If it's the first time you join to Azure DevOps you will need to create an organization, if it's for personal usage you can keep your username as organization.

3. Once you have been configured your organization a new project has been created as default, let's ignore that project and let's create a new one with the following attributes:

    - Name: WBD
    - Visibility: Private
    - Advanced -> Version Control: Git
    - Advanced -> Work item process: Scrum

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-project.png" />
</div>

Great! now you are able to see your new project called: WBD.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-dashboard.png" />
</div>

#### Cloning the Repo

First, we need to upload our bot code to the Azure DevOps Git repo. To perform this action click in <b>Repos->Files</b>, then click in the button <b>Generate Git credentials</b>.

Pick an alias and a password, this alias and password will be used to perform all Git operations via terminal, finally click in <b>Save Git Credentials</b>.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-credentials.png" />
</div>

Now, we need to perform the following tasks:

- Clone the remote repository in the local repository path.
- Create a new source folder in the local repository path.
- Copy to the local repository path your bot's code from `source\1. start\` or `source\2. completed\` depending if you complete or not the laboratory.
- Commit the changes in the local repository.
- Push the changes in the remote repository.

Steps:

1. Open a terminal in the local repository path.

```bash
C:/GitHub>git clone https://technical-poets@dev.azure.com/technical-poets/WBD/_git/WBD
```

2. Introduce your Azure DevOps credentials or your Git credentials.

3. After clone your remote repository you will be able to see the folder WBD inside GitHub.

4. Copy and paste the files from `source\1. start\` or `source\2. completed\` into C:\GitHub\WBD\, the files  depends if you complete or not the laboratory.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-files.png" />
</div>

5. In the terminal write:

```bash
C:/GitHub/WBD>git add .
```

```bash
C:/GitHub/WBD>git commit -a -m "Initial commit"
```

```bash
C:/GitHub/WBD>git push
```

6. Once you pushed the changes to the remote repo, you will see the following message:

```bash
Enumerating objects: 27, done.
Counting objects: 100% (27/27), done.
Delta compression using up to 4 threads.
Compressing objects: 100% (23/23), done.
Writing objects: 100% (27/27), 11.24 KiB | 1.41 MiB/s, done.
Total 27 (delta 1), reused 0 (delta 0)
remote: Analyzing objects... (27/27) (7 ms)
remote: Storing packfile... done (54 ms)
remote: Storing index... done (66 ms)
To https://dev.azure.com/technical-poets/WBD/_git/WBD
 * [new branch]      master -> master
 ```

7. If you return to Azure DevOps portal in <b>Repos->Files</b>, you will be able to see the pushed files with Git.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-filesportal.png" />
</div>

#### Creating the CI Pipeline

Now, it's time to configure the continuous integration pipeline. To perform this action click in <b>Pipelines->Builds</b> and then click `New pipeline`.

Steps:

1. Select the option: `Use the visual designer to create a pipeline without YAML`.

2. Select the current Azure Repos Git, the WBD team project, WBD repository and master branch then click continue.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-azurerepos.png" />
</div>

3. Choose the following template: ASP.NET and click Apply.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-citemplate.png" />
</div>

4. Now, you will be able to see your CI pipeline complete, in this scenario you only need to `Enable continuous integration` in the Triggers tab and click save to save the CLI template.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-citemplate2.png" />
</div>

5. Finally, you can `queue a new build` or `submit a new change in the code` to verify if the code is successfully compiled.

6. Click again in n <b>Pipelines->Builds</b> and you will see the current build in execution, if you want to see the details click in the commit name.

7. Let's validate that the build is compiled successfully.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-build.png" />
</div>

8. Awesome, at this moment you have your first bot artifact ready to be deployed.

#### Creating the Web App Bot in Azure Portal

Go to https://portal.azure.com/ and get successfully sign-in with your Employee or Microsoft account.

Steps:

1. Create a Web App Bot resource with the following configuration:

    - Bot Name: wbd(uniqueid) e.g. wbd017.
    - Subscription: your subscription.
    - Resource Group: Create new with the same bot name, e.g. wbd017.
    - Location: region desired, e.g. West US.
    - Bot Template: SDK 3 / C# / Basic (we are going to override this bot with our code using SDK 4, I'm not using SDK 4 directly because it has a bot configuration file that is out of the scope of this walkthrough).
    - App Service Plan: Create new with the same bot name and location, e.g. App service plan name: wbd017 and location: West US.
    - Azure Storage: Create new.
    - Application Insights: Off.
    - Microsoft App Id and Password: Auto create.

2. Let's validate that the resources have been created successfully in the Azure portal.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-deployment.png" />
</div>

#### Creating the CD Pipeline

Now, it's time to configure the continuous deployment pipeline. To perform this action click in <b>Pipelines->Releases</b> and then click `New pipeline`.

Steps:

1. Choose the following template: Azure App Service deployment and click Apply.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-cdtemplate.png" />
</div>

2. Rename the Stage from `Stage 1` to `Azure Web App Bot`.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-stage.png" />
</div>

3. Click in `Add an artifact` option and select the CI pipeline artifact configured previously then specify a source alias and click Add.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-artifact.png" />
</div>

4. Select the continuous deployment trigger and enable the option to create a release everytime there is a new build.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-triggerrelease.png" />
</div>

5. Select Tasks tab and configure the following settings.

    - Azure Subscription (click Authorize to stablish connection with the account).
    - App Type: Web App
    - App service name: name of your Web App Bot (e.g. wbd017)

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-cdtemplateconfig.png" />
</div>

6. Click in the add a new task icon and search: `Set App Service: Set App settings`. If it's the first time you use the plugin from the marketplace you need to install it, save you pipeline and then refresh the Azure DevOps page to be able to added in the pipeline.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-appsettings.png" />
</div>

7. Configure the Set App settings task with the following values.

    - TranslatorTextAPIKey='AZURE_TRANSLATOR_KEY'
    - BotVersion='BOT_VERSION'
    - LuisName01='LUIS_NAME (e.g. Reminders)'
    - LuisAppId01='LUIS_APPLICATION_ID'
    - LuisAuthoringKey01='LUIS_AUTHORING_KEY'
    - LuisEndpoint01='LUIS_ENDPOINT'

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-appsettings2.png" />
</div>

8. Click in the add a new task icon and search: `Azure App Service Manage` and configure it to restart the app service.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-restart.png" />
</div>

9. Save the release pipeline and queue a new build, if everything is correctly configured you will be able to see the deployment completed successfully in <b>Pipelines->Releases</b>.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-deploymentsuccessfully.png" />
</div>

10. You can validate the correct settings directly in the Azure portal.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-azureappsettings.png" />
</div>

11. Congratulations!, you can now check your bot in the webchat window in the Azure portal.

<div style="text-align:center">
    <img src="http://rcervantes.me/images/walkthrough-bot-dotnet-azuredevops-webchat.png" />
</div>
