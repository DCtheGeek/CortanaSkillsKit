---
title: Using AdaptiveCards in your skill
description: Learn how to use adaptive cards in your bot-based skill.
label: Conceptual

ms.assetid: A7CD987E-5DD1-42EA-A436-49D4E8327365
ms.date: 06/27/2019
ms.topic: article

keywords: cortana
---

# Using AdaptiveCards in your Cortana skill

Cards are interface elements that you can use to enhance the user experience in your Cortana skill. An adaptive card is the most versatile display card. It's customizable, and can include any combination of text, speech, images, buttons, and input fields.

As with all cards, you can only use them when Cortana is running on a device with a display. See [Determine Cortana's device type](./cortana-device-type.md) for details on how to get device information.
 
## AdaptiveCards  

Adaptive cards provide the following options.  

|     |     |
| --- | --- |
|**Input controls** | Add input controls for text, date, number, time, toggle switch, and choice set.  |
|**Richer text** | Text in your card is not limited to title, subtitle, and text fixed formats. Use a variety of font sizes, formats, and colors. |
|**A single open card exchange format** | Use your existing cards in a common and consistent way and extend your cards with rich controls using a common schema.  |

AdaptiveCards use the open card exchange format. This format enables you to specify user interface content for all cards in your skill in a common and consistent way. You describe the content as a simple JSON object. The JSON content is natively displayed by the skill and automatically adapts to the look and feel of your skill.

>[!NOTE]
>Cortana currently supports AdaptiveCards version 1.0.

AdaptiveCards include elements, containers, actions, and inputs. A basic adaptive card includes:

* an adaptive card root object,
* an adaptive card body, which includes the elements of your card, and
* actions for your adaptive card, which are typically displayed in an action bar at the bottom of your card.  

## AdaptiveCards Designer

The [AdaptiveCards Designer](https://adaptivecards.io/designer) provides an interactive card builder where you can see the resulting card JSON data.

## Create an AdaptiveCard
You can create adaptive cards using `proxy pattern` helper classes in the [SDK](/adaptive-cards/), or by directly using JSON (check out the [Schema Explorer](https://adaptivecards.io/explorer/) for details).

>[!IMPORTANT]
> 1. The speak object of an adaptive card must be copied to the Message for Cortana to speak the text.
> 1. The speak object text must be wrapped in SSML `<speak>` tags. (See the [Speech Synthesis Markup Language (SSML) reference](./speech-synthesis-markup-language.md).)

# [.NET](#tab/dotnet)

### Create using .NET

1. Install the `AdaptiveCards` NuGet package.
1. Specify the elements of your card in code.
1. Add the card to your Cortana skill as an attachment.

The following code adds an adaptive card to a Cortana skill response for Bot Framework version 4.

 ```csharp
 // make a response
 var response = turnContext.Activity.CreateReply();

 // create a Card and add elements
 AdaptiveCard card = new AdaptiveCard();
 card.Body.Add(new AdaptiveTextBlock()
     {
         Text = "This is a test",
         Weight = TextWeight.Bolder
         Size = TextSize.Medium,
     }
 );

 // add card as attachment
 response.Attachments.Add(card.ToAttachment());

 // send the response
 await turnContext.SendActivityAsync(response, cancellationToken: cancellationToken);

 ```  

# [JavaScript](#tab/js)

### Create using JavaScript

1. Install the `adaptivecards` [NPM package](/adaptive-cards/sdk/rendering-cards/javascript/getting-started#install) (optional)
1. Use JSON to build your card in code, or the proxy object from the `adaptivecards` package
1. Add the card to your skill as an attachment.

 ```javascript
// In JavaScript, JSON is integrated
// Create the Card from JSON
const card = CardFactory.adaptiveCard( {
  "contentType": "application/vnd.microsoft.card.adaptive",
  "content": {
    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
    "type": "AdaptiveCard",
    "version": "1.0",
    "body": [{
      "type": "TextBlock",
      "text": "This is a test",
      "size": "medium",
      "weight": "bolder"
       }]
    }
  } );
  
// Send the card  
await turnContext.sendActivity( {
 text: 'Here is an Adaptive Card:',
 speak: card.content.speak || 'Sorry, this card does not contain speech.',
 attachments: [ card ] 
 } );
 ```

---

## Respond to AdaptiveCards
If you look at the [Calendar reminder](https://adaptivecards.io/samples/CalendarReminder.html) example on the AdaptiveCards website, you'll note that it has two `Action.Submit` buttons, one for the _Snooze_ response, and one for the _I'll be Late_ response.  For _Snooze_, there is a `Input.ChoiceSet` with standard values of 5, 15, and 30 minutes.

In our example, Cortana will say, _"Your meeting 'Adaptive Card design session' is starting at 12:30pm. Do you want to snooze or do you want to send a late notification to the attendees?"_  At the same time, she will display this card:

 ![Sample card](../media/images/calendar_reminder.png)  

Cortana skills should be designed for voice first, so consider this code that supports spoken replies like
- I'll be **late**
- **Snooze** for 10 minutes

based on simple keyword search (matches in bold).

You will receive a JSON `value` attached to the response message with the `data` properties of the `Action.Submit` buttons, along with any `Input` fields with their `id` and `value` fields.  Given this example, if the user clicks the _Snooze_ button, they'll see:

```
 "value": {
      "x": "snooze",  <-- the Action.Submit data
      "snooze": "5"   <-- the Input.ChoiceSet id and selected value
    }
```

>[!IMPORTANT]
>Cortana responds differently depending on how the user responds. If the user clicks the button, Cortana returns the value property. If the user speaks the response, Cortana sends the text property.

The example returns data in the JSON `value` because the user pressed the button on the card. If the response is spoken, there will be a `text` response, but no `value`.

You should ignore the `text` property on the message if a `value` is present.

Your code must handle both cases in order to handle conversations correctly.

# [C#](#tab/cs)

### Respond in C#

```C#
...
        // match 1 or 2 digits and white space and "minute"
        private static Regex regexMinutes = new Regex(@"(\d{1,2})\s+minute", RegexOptions.IgnoreCase);
...
                var message = turnContext.Activity;
                var response = turnContext.Activity.CreateReply(); 

                string sValue = "unknown";

                if (message.Value != null)
                {
                    // Got an Action Submit
                    dynamic value = message.Value;
                    string xValue = (string)value.SelectToken("x");
                    if (xValue.Equals("snooze"))
                    {
                        string snoozeValue = (string)value.SelectToken("snooze");
                        response.Text = "You clicked \"Snooze\" for " + snoozeValue + " minutes.";
                        response.Speak = response.Text;
                    }
                    else if (xValue.Equals("late"))
                    {
                        response.Text = "You clicked \"I'll be late.\"";
                        response.Speak = response.Text;
                    }
                    else
                    {
                        response.Text = "Unsupported input detected.";
                        response.Speak = response.Text;

                        sValue = value.ToString();
                        Trace.WriteLine(sValue);
                    }
                }
                else
                {
                    // Got a Voice/Text response - check for keywords
                    string sText = message.Text.ToUpper(); // as Contains doesn't allow case insensitive
                    if (sText.Contains("SNOOZE"))
                    {
                        Match m = regexMinutes.Match(sText);
                        if (m.Success)
                        {
                            Group g = m.Groups[1];
                            string snoozeValue = g.Value;
                            response.Text = "You said \"Snooze\" for " + snoozeValue + " minutes.";
                            response.Speak = response.Text;
                        }
                        else
                        {
                            response.Text = "You said \"Snooze\". I'll snooze for the default 5 minutes.";
                            response.Speak = response.Text;
                        }
                    }
                    else if (sText.Contains("LATE"))
                    {
                        response.Text = "You said \"I'll be late.\"";
                        response.Speak = response.Text;
                    }
                    else
                    {
                        response.Text = "I am not sure what you mean.";
                        response.Speak = response.Text;
                    }
                }

                await turnContext.SendActivityAsync(response, cancellationToken: cancellationToken);
```

# [JavaScript](#tab/js2)

### Respond in JavaScript

```JavaScript
        let message = turnContext.activity;
        var response = {};

        if (message.value) {
            // Got an Action Submit payload
            let xValue = message.value.x;
            if (xValue === 'snooze') {
                let snoozeValue = message.value.snooze;
                let msg = `You clicked "Snooze" for ${snoozeValue} minutes.`;
                response.text = msg;
                response.speak = msg;
            } else if (xValue === 'late') {
                const msg = 'You clicked "I\'ll be late."';
                response.text = msg;
                response.speak = msg;
            } else {
                const msg = 'Unsupported input detected.';
                response.text = msg;
                response.speak = msg;
            }
        } else {
            // Got a Voice/Text response - check for keywords
            let text = message.text.toUpperCase();
            if (text.includes('SNOOZE')) {
                // match 1 or 2 digits and white space and "minute"
                const regexMinutes = new RegExp('(\\d{1,2})\\s+minute', 'i');
                var match = regexMinutes.exec(text);
                if (match && match.constructor === Array && match.length == 2) {
                    let msg = `You said "Snooze" for ${match[1]} minutes.`;
                    response.text = msg;
                    response.speak = msg;
                } else {
                    const msg = 'You said "Snooze". I\'ll snooze for the default 5 minutes.';
                    response.text = msg;
                    response.speak = msg;
                }
            } else if (text.includes('LATE')) {
                const msg = 'You said "I\'ll be late."';
                response.text = msg;
                response.speak = msg;
            } else {
                const msg = 'I am not sure what you mean.';
                response.text = msg;
                response.speak = msg;
            }
        }
        await turnContext.sendActivity(response);
```

---

## More information  

* For more information about adaptive cards, visit the  [adaptivecards.io](https://adaptivecards.io) page.  
* For more information about Bot Framework cards, see the [Send an Adaptive Card](/azure/bot-service/bot-builder-howto-add-media-attachments?tabs=csharp&view=azure-bot-service-4.0#send-an-adaptive-card) section on the [Add media to messages](/azure/bot-service/bot-builder-howto-add-media-attachments?tabs=csharp&view=azure-bot-service-4.0) page.
* For more info on the examples referenced on this page, please visit [cortana-skills-samples on GitHub](https://github.com/Microsoft/cortana-skills-samples)