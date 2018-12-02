## Programming the Bot

#### Clone the project from GitHub repo

`git clone https://github.com/rcervantes/walkthrough-bot-dotnet.git`

#### Using Visual Studio 2017 (any version)

1. Based on the webinar explanation go to `source\1. start\` folder and open the WBD.sln file in Visual Studio 2017.

2. Wait until nuget packages are being reinstalled.

3. Build the WBD project (the project should be successfully compiled but is not ready to be executed).

#### Using Visual Studio Code (any version)

1. Open Visual Studio Code and open the following folder `source\1. start\WBD\`.

2. Open a terminal pointing the folder `source\1. start\WBD\` folder and run the following command: <b>dotnet build</b>, wait until nuget packages are being reinstalled and the project finishes the build process (the project should be successfully compiled but is not ready to be executed).

#### Translator text configuration

Go to https://portal.azure.com/ and get successfully sign-in with your Employee or Microsoft account.

Steps:

1. Create a Translator Text resource with the following configuration:

    - Name: translator(uniqueid) e.g. translator017.
    - Subscription: your subscription.
    - Pricing tier: S1 (Pay as you go).
    - Resource Group: Create new with the same resource name, e.g. translator017.

2. Once the resource has been deployed go to the resource and click in <b>Resource Management->Keys</b> and take note of the `Key 1` in a notepad (we are going to use this information later to configure the bot).

#### LUIS configuration

1. Go to https://www.luis.ai/ and get successfully sign-in with your Employee or Microsoft account.

2. Once you are signed-in, go to My Apps and click `Import new app`, select the file: Reminders.json located in the folder `source\3. models\` and finally click done.

3. A new LUIS application has been created (this application contains three intents: Calendar.Add,Calendar.Find and None, each intent has been filled with a bunch of utterances as an example).

4. Go to Reminders app and click on Train button, once you have trained the model click Publish to make public the API service.

5. Once the service has been trained and published go to manage and take note of the `Application ID (from Application Information tab)`, `Authoring Key (from Keys and Endpoints tab)`, `Endpoint (from Keys and Endpoints tab)` in a notepad (we are going to use this information later to configure the bot).

6. Everytime you add, remove or update utterances a new train and publish process is required to train and expose the latest version.

#### Bot configuration

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

#### Getting dirty (let's code!)

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

#### Running the bot

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

#### Master code (complete solution)

In case you want to review your code with the complete solution you can follow the previous configuration steps using the complete solution located in: `source\2. completed\WBD\`.