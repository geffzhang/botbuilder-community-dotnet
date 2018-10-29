## Alexa Adapter for Bot Builder v4 .NET SDK

### Build status
| Branch | Status | Recommended NuGet package version |
| ------ | ------ | ------ |
| master | [![Build status](https://ci.appveyor.com/api/projects/status/b9123gl3kih8x9cb?svg=true)](https://ci.appveyor.com/project/garypretty/botbuilder-community) | [![NuGet version](https://img.shields.io/badge/NuGet-1.0.100-blue.svg)](https://www.nuget.org/packages/Bot.Builder.Community.Adapters.Alexa/) |

### Description

This is part of the [Bot Builder Community Extensions](https://github.com/garypretty/botbuilder-community) project which contains various pieces of middleware, recognizers and other components for use with the Bot Builder .NET SDK v4.

The Alexa Adapter allows you to add an additional endpoint to your bot for Customer Alexa Skills. The Alexa endpoint can be used
in conjunction with other channels meaning, for example, you can have a bot exposed on out of the box channels such as Facebook and 
Teams, but also via an Alexa Skill.

Incoming Alexa Skill requests are transformed, by the adapter, into Bot Builder Activties and then when your bot responds, the adapter transforms the outgoing Activity into an Alexa response.

The adapter supports a broad range of capabilities for Alexa Skills, including;

* Support for voice based Alexa bots
* Support for the available display directives for Echo Show / Spot devices, with support for the new Fire Tablets coming very soon
* Support for Alexa Cards
* Support for Audio / Video directives
* Integration libraries (similar to those for the Bot Framework Adapter), allowing for the bot to also have its own middleware pipeline for Alexa
* TurnContext extensions allowing the developer to;
    * Send Alexa Progressive Updates
    * Explicitly define and attach an Alexa Card to the response (alternatively Bot Builder Hero / Signin cards can be automatically converted if attached to the outgoing activity)
    * Specify Alexa RePrompt speech / text
    * Add to / access Alexa Session Attributes (similar to TurnState in Bot Builder SDK)
    * Check if a device supports audio or audio and display
    * Retrieve the raw Alexa request
    * A new method to allow the user to call the relevant API and access user details (with skill permission), such as address
* Validation of incoming Alexa requests (required for certification)
* Middleware to automatically translate incoming Alexa requests into different types of activities


### Installation

Available via NuGet package [Bot.Builder.Community.Adapters.Alexa](https://www.nuget.org/packages/Bot.Builder.Community.Adapters.Alexa/)

Install into your project using the following command in the package manager;
```
    PM> Install-Package Bot.Builder.Community.Adapters.Alexa
```

### Usage

#### Adding the adapter and skills endpoint to your bot

Currently there are integration libraries available for WebApi and .NET Core available for the adapter.
When using the integration libraries a new endpoint for your Alexa skill is created at '/api/skillrequests'. 
e.g. http://www.youbot.com/api/skillrequests.  This is the endpoint that you should configure within the Amazon Alexa
Skills Developer portal as the endpoint for your skill.

##### WebApi

When implementing your bot using WebApi, the integration layer for Alexa works the same as the default for Bot Framework.  The only difference being in your BotConfig file under the App_Start folder you call MapAlexaBotFramework instead;

```cs
    public class BotConfig
    {
        public static void Register(HttpConfiguration config)
        {
            config.MapAlexaBotFramework(botConfig => { 
				botConfig.AlexaBotOptions.AlexaOptions.ShouldEndSessionByDefault = true;
                botConfig.AlexaBotOptions.AlexaOptions.ValidateIncomingAlexaRequests = false;
			});
        }
    }
``` 

##### .NET Core

An example of using the Alexa adapter with a bot built on Asp.Net Core. Again, The implementation of the 
integration layer is based upon the same patterns used for the Bot Framework integration layer. 
In Startup.cs you can configure your bot to use the Alexa adapter using the following;

```cs
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddAlexaBot<EchoBot>(options =>
            {
                options.AlexaOptions.ValidateIncomingAlexaRequests = true;
                options.AlexaOptions.ShouldEndSessionByDefault = false;
            });
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
        {
            app.UseDefaultFiles()
                .UseStaticFiles()
                .UseAlexa();
        }
```

#### Default Alexa Request to Activity mapping

When an incoming request is receieved, the activity sent to your bot is comprised of the following values;

* **Channel ID** : "alexa"
* **Recipient Channel Account** : Id = Application Id from the Alexa request, Name = "skill"
* **From Channel Account** : Id = User Id from the Alexa request, Name = "user"
* **Conversation Account** : Id = "{Alexa request Application Id}:{Alexa Request User Id}"
* **Type** : Request Type from the Alexa request. e.g. IntentRequest, LaunchRequest or SessionEndedRequest
* **Id** : Request Id from the Alexa request
* **Timestamp** : Timestamp from the Alexa request
* **Locale** : Locale from the Alexa request

For incoming requests of type IntentRequest we also set the following properties on the Activity

* **Value** : DialogState value from the Alexa request

For incoming requests of type SessionEndedRequest we also set the following properties on the Activity

* **Code** : Reason value from the Alexa request
* **Value** : Error value from the Alexa request

The entire body of the Alexa request is placed into the Activity as Channel Data, of type AlexaRequestBody.

#### Default Activity to Alexa Response mapping

The Alexa adapter will send a response to the Alexa skill request if the outgoing activity is of type MessageActivity or EndOfConversation activity.

If the actvity you send from your bot is of type EndOfConversation then a response is sent indicating that the session should be ended, by setting the the ShouldEndSession flag on the ALexa response to true.

If the activity type you send from your bot is of type MessageActivity the following values are mapped to an Alexa response object;

* **OutputSpeech Type** : Set to 'SSML' if activity.Speak is not null. Set to 'PlainText' if the activity.Text property is populated but the activity.Speak property is not.
* **OutputSpeech SSML** : Populated using the value of activity.Speak if it is not null.
* **OutputSpeech Text** : Populated using the value of activity.Text if it is not null.

* **ShouldEndSession** : Defaults to false. However, setting the InputHint property on the activity to InputHint.IgnoringInput will set this value to true and end the session.

#### TurnContext Extension Methods

##### Session Attributes

Alexa Skills use Session Attributes on the request / response objects to allow for values to be persisted accross turns of a conversation.  When an incoming Alexa request is receieved we place the Session Attributes on the request into the Services collection on the TurnContext.  We then provide an extension method on the context to allow you to add / update / remove items on the Session Attributes list. Calling the extension method AlexaSessionAttributes returns an object of type Dictionary<string, string>. If you wanted to add an item to the Session Attributes collection you could do the following;

```cs 
    context.AlexaSessionAttributes.Add("NewItemKey","New Item Value");
```

##### Progressive Responses

Alexa Skills allow only a single primary response for each request.  However, if your bot will be running some form of long running activity (such as a lookup to a 3rd party API) you are able to send the user a holding response using the Alexa Progressive Responses API, before sending your final response.

To send a Progressive Response we have provided an extension method on the TurnContext called AlexaSendProgressiveResponse, which takes a string parameter which is the text you wish to be spoken back to the user. e.g.

```cs
    context.AlexaSendProgressiveResponse("Hold on, I will just check that for you.");
```

The extension method will get the right values from the incoming request to determine the correct API endpoint / access token and send your Progressive response for you.  The extension method will also return a HttpResponseMessage which will provide information as to if the Progressive Response was send successfully or if there was any kind of error.

***Note: Alexa Skills allow you to send up to 5 progressive responses on each turn.  You should manage and check the number of Progressive Responses you are sending as the Bot Builder SDK does not check this.*** 


##### Get entire Alexa Request Body

We have provided an extension method to allow you to get the original Alexa request body, which we store on the ChannelData property of the Activity sent to your bot, as a strongly typed object of type AlexaRequestBody.  To get the request just call the extension method as below;

```cs
    AlexaRequestBody request = context.GetAlexaRequestBody();
```

***Note: If you call this extension method when the incoming Activity is not from an Alexa skill then the extension method will simply return null.*** 


##### Add Directives to response

Add objects of type IAlexaDirective to a collection used when sending outgoing requests to add directives to the response.  This allows you to do things like 
controlling the display on Echo Show / Spot devices.  Classes are included for Display and Hint Directives.

```cs
	dialogContext.Context.AlexaResponseDirectives().Add(displayDirective);
```

##### Check if device has Display or Audio Player support

```cs 
    dialogContext.Context.AlexaDeviceHasDisplay()

	dialogContext.Context.AlexaDeviceHasAudioPlayer()
```

#### Alexa Middleware

TO BE COMPLETED


#### Automated Translation of Bot Framework Cards into Alexa Cards

The Alexa Adapter supports sending Bot Framework cards of type HeroCard, ThumbnailCard and SigninCard as part of your replies to the Alexa skill request.

The Alexa adapter will use the first card attachment by default, unless you have disabled this using the AlexaBotOptions property ConvertFirstActivityAttachmentToAlexaCard.

* **HeroCard and ThumbnailCard** : 

 * Alexa Card Small Image URL = The first image in the Images collection on the Hero / Thumbnail card
 * Alexa Card Large Image URL = If a second image exists in the Images collection on the Hero / Thumbnail card this will be used. If no second image exists then this is null.
 * Alexa Card Title = Title property of the Hero / Thumbnail card
 * Alexa Card Content = Text property on the Hero / Thumbnail card

***Note: You should ensure that the images you use on your HeroCard / Thumbnail cards are the correct expected size for Alexa Skills responses.***

* **SigninCard** : If a SignInCard is attached to your outgoing activity, this will be mapped as a LinkAccount card in the Alexa response.


#### Alexa Show / Spot Display Support

Display Directives to support devices with screens, such as the Echo Show and Spot, can be added to your bots response.  A class for each of the currently supported templates 
exists within the Alexa.Directives namespace.  A context extension method allows you to add directives to the services collection which will then be used by the Alexa Adapter 
when processing outgoing activities and will add the appropriate JSON to the response.

***Note: You should also use the AlexaDeviceHasDisplay() extension method on the ITurnContext object to check if the Alexa device that has sent the incoming 
request has a display - this is because if you send a display directive to a device without a display it will cause an error and not simply be ignored.***

``` cs

            var displayDirective = new DisplayDirective()
            {
                Template = new DisplayRenderBodyTemplate1()
                {
                    BackButton = BackButtonVisibility.HIDDEN,
                    Title = "Claim Update",
                    TextContent = new TextContent()
                    {
                        PrimaryText = new InnerTextContent()
                        {
                            Text = "<font size=\"7\"><b>Good news!</b></font>",
                            Type = TextContentType.RichText
                        },
                        SecondaryText = new InnerTextContent()
                        {
                            Text = "<br/><font size=\"3\">This is your Secondary Text",
                            Type = TextContentType.RichText
                        },
                        TertiaryText = new InnerTextContent()
                        {
                            Text = "This is tertiary text - this time it is plain text",
                            Type = TextContentType.PlainText
                        }
                    },
                    Token = "",
                    BackgroundImage = new Image()
                    {
                        ContentDescription = "test",
                        Sources = new ImageSource[]
                                {
                                    new ImageSource()
                                    {
                                        Url = "https://www.yourimageurl.com/background.jpg",
                                        WidthPixels = 1025,
                                        HeightPixels = 595
                                    }
                                }
                    }
                }
            };

            if (dialogContext.Context.AlexaDeviceHasDisplay())
            {
                dialogContext.Context.AlexaResponseDirectives().Add(displayDirective);
            }

```