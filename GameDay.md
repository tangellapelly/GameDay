## Building a gas stations chatbot with AWS Lex, Lambda and HERE services:

![alt text](screenshots/chatbot.png "testing the chatbot")

In this tutorial, we will deploy a small chatbot with AWS Lex and HERE servies to help users find the gas stations near by a specified location. We will explore the combination of AWS services, HERE API services and how they can be stitched together to create a powerful, scalable and secure application. To complete this tutorial, you will need: An AWS Account, basic knowledge of the AWS UI, account with developer.here.com ,and some basic understanding of node/Javascript.

# Step 1: Deploy Lex bot

Login to your AWS console, find Amazon Lex and click on it. Now, click on the create button to get started and then on the next screen, click Custom Bot, fill in the details as shown in the screenshot below.

![alt text](screenshots/step1.png "Step 1: Deploy Lex bot")

Once you fill in the all the required details click on *Create*
button to navigate to the next screen. 

![alt text](screenshots/step2.png "Create Intent")

On the next screen you will find Create Intent. An Intent is a objective or action the user wants to achieve. Think of an intent as the specific reason a user would want to send a message to your bot. Lets create an intent and call it gasStationNearBy. 

![alt text](screenshots/step3.png "Create Intent")

Once we’ve done that, we’re greeted with another screen with a whole bunch of input options. Lets focus on Slots first.

![alt text](screenshots/step4.png "Slots")

Slots are the variables of Lex. If someone was ordering a pizza, the Slots to include would be things like {SIZE}, {PIZZA_TYPE} and {DELIVERY_ADDRESS}. For our chatbot, we will have one slot. {gpsCodes}. There is a lot to discuss about slots and different slot types. Essentailly there are two types of slot available custom slots and builtin slots. For our chatbot we will be creating the custom slot. <a href="https://docs.aws.amazon.com/lex/latest/dg/howitworks-builtins-slots.html">So, I encourage you to see more information about slots on the amazon amazon docs</a>.

To create a custom slot, go ahead added new slot type from the left hand side menu, you will find a dialog box which will have create slot type(As shown in the screenshot below)

![alt text](screenshots/step5.png "Creating Custom Slots")

Once you click on the *create slot type* you will be greeted with a bunch of options to fill in. You can follow below screenshot to fill and go head and click on the *Add slot to intent*

![alt text](screenshots/step6.png "Creating Custom Slots")

Once the slot successfully saved, you can find the *gpsCodes* slot under the custom slot as shown in the below screenshot.

![alt text](screenshots/step7.png "selecting the custom slot to the intent")

Now that we have added the out custom slot, let us go ahead and setup the slot for our intent. As you can see in the following screenshot, we have to give a name to our custom slot, from the slot type dropdown please select gpsCodes under the *Custom slot types* and then give a prompt like so _which gps coordinates?_ Click on the add button under the settings to apply the slot to the current intent. Finally, click on the *Save Intent* button located on bottom of the page.

![alt text](screenshots/step8.png "applying the slot to the intent")

We need to add in variations of that utterance you can think of,but keep it relatively short to 2–3 different variations. Lets add the following utterances

* gas stations near {gpsCodes}
* petrol stations near {gpsCodes}
* fuel stations near {gpsCodes}

![alt text](screenshots/step9.png "Adding uttetances")

Notice for Fulfillment, you have the option to either send the intent to a Lambda function or return parameters to client. For now, lets click Return Parameters to Client and click save. We’re going to check on our bot before moving over to Amazon Lambda and creating the business logic of our chatbot. Scroll down and save your intent.

![alt text](screenshots/step10.png "Adding Fulfillment")

And now for a bit of fun! On the top right of the dashboard, click Build. This will initialize your chatbot and allow you to begin testing it. A chat widget should pop up on the right side of the screen. Try it out!

![alt text](screenshots/step11.png "Building and testing the bot")
Awesome, our Lex chatbot is almost ready to go. Our next step is to attach a Lambda function to it.

Lambda will simply receive the slots and their values and return them in a way that Lex understands

## Step 2: Deploy a Lambda Function

In your AWS console, go to Amazon Lambda and select Build a Function from Scratch. In the next screen, you will have the option of building your custom function or selecting a blueprint. The blueprints are great examples and I highly recommend checking them out to enhance your knowledge. For our purposes, Select Author From Scratch and fill out the form with your function name, runtime environment (Node.js 10.x) and select the *create function* button to create the lambda function.


![alt text](screenshots/step12.png "creating lambda function")

On this next Screen you will be treated to your lambda function, the designer tool, and some dashboard items for testing and monitoring your function. The following is the lambda function that we wanted to create:

```javascript
'use strict';
const axios = require('axios')     
function close(sessionAttributes, fulfillmentState, message) {
    return {
        sessionAttributes,
        dialogAction: {
            type: 'Close',
            fulfillmentState,
            message,
        },
    };
}
 
// --------------- Events -----------------------
 
async function dispatch(intentRequest, callback) {
    console.log(`request received for userId=${intentRequest.userId}, intentName=${intentRequest.currentIntent.name}`);
    const sessionAttributes = intentRequest.sessionAttributes;
    const slots = intentRequest.currentIntent.slots;
    const city = slots.city;
 
    if(slots.gpsCodes){
        
          const gpsCodes = slots.gpsCodes;
    const codes = gpsCodes.split(',');
       // API to get the list of gas stations
              let gasStation = axios.get(`https://places.demo.api.here.com/places/v1/discover/search?at=${codes[0].trim()},${codes[1].trim()}&q=petrol-station&app_id=MMRyT9PioGx6DeImyPie&app_code=SB7YD1dqPH40vz-lSJE19g`, {}).then((data)=>{
                 
                let gasStationNames= data.data.results.items.map((it)=>{
                    
                    return ` *${it.title}* is at _${it.vicinity.replace("<br/>"," ")}_`+"\n"
                })
                const reducer = (accumulator, currentValue) => accumulator + currentValue;

                callback(close(sessionAttributes, 'Fulfilled',
                {'contentType': 'PlainText', 'content': `Okay, Here are the list of gas stations ${"\n"}${gasStationNames.reduce(reducer)}`}));
                
            }).catch((e)=>{
        console.log(e)
    })  
    } else {
         // API to convert the address to lat and lng 
    const res = await axios.get(`https://geocoder.api.here.com/6.2/geocode.json?app_id=MMRyT9PioGx6DeImyPie&app_code=SB7YD1dqPH40vz-lSJE19g&searchtext=${city}`, {}).then((res)=>{
            let lat = res.data.Response.View[0].Result[0].Location.DisplayPosition.Latitude;
            let lng = res.data.Response.View[0].Result[0].Location.DisplayPosition.Longitude
            
            // API to get the list of gas stations
              let gasStation = axios.get(`https://places.demo.api.here.com/places/v1/discover/search?at=${lat},${lng}&q=petrol-station&app_id=MMRyT9PioGx6DeImyPie&app_code=SB7YD1dqPH40vz-lSJE19g`, {}).then((data)=>{
                 
                let gasStationNames= data.data.results.items.map((it)=>{
                    
                    return ` *${it.title}* is at _${it.vicinity.replace("<br/>"," ")}_`+"\n"
                })
                const reducer = (accumulator, currentValue) => accumulator + currentValue;

                callback(close(sessionAttributes, 'Fulfilled',
                {'contentType': 'PlainText', 'content': `Okay, Here are the list of gas stations near by ${city}${"\n"}${gasStationNames.reduce(reducer)}`}));
                
            })
           
    }).catch((e)=>{
        console.log(e)
    })
   
   
    }
    
}




// --------------- Main handler -----------------------
 
// Route the incoming request based on intent.
// The JSON body of the request is provided in the event slot.
exports.handler = (event, context, callback) => {
    try {
        dispatch(event,
            (response) => {
                callback(null, response);
            });
    } catch (err) {
        callback(err);
    }
};
```


The first time you hit Test, you will be asked to supply a JSON snippet describing what the test input should be.

``` json
{
  "messageVersion": "1.0",
  "invocationSource": "FulfillmentCodeHook",
  "userId": "user-1",
  "sessionAttributes": {},
  "bot": {
    "name": "HerePlacesBot",
    "alias": "$LATEST",
    "version": "$LATEST"
  },
  "outputDialogMode": "Text",
  "currentIntent": {
    "name": "HerePlacesBot",
    "slots": {
      "gpsCodes": "18.000055,79.588165"
    },
    "confirmationStatus": "None"
  }
}
```

Here is the <a target="_blank" href="https://heremaps.s3.amazonaws.com/getPetrolStations-9cf1d5f0-1ce7-4da0-9a44-e9ace1954d49.zip">link</a> to download the Lambda deployment package. Once you download the zip file, upload into the AWS Lambda function code dashboard as shown in the below screenshot.

![alt text](screenshots/step13.png "creating lambda function")

Now, select the *Upload a zip file* option from the dropdown to upload the downloaded zip file. 

![alt text](screenshots/step14.png "creating lambda function")

Now that we have created out Lambda function, it time to integrate it with Amazon Lex. Head over to the lex dashboard and select the bot that we have already created to integrate the Lambda function.


![alt text](screenshots/step15.png "creating lambda function")

Go the chatbot setting and under the Fullfillment select _AWS Lambda function_ and choose the getGasStationsNearBy function. It will ask for the permissions, go ahead and click on _OK_


![alt text](screenshots/step16.png "creating lambda function")

Finally, click on the _Save Intent_ button towards the bottom of the page to save all the settings.

## Step 3: Testing the chatbot

Its the fun part now! In order to test our chatbot, click on the _build_ button on the top right of the dashboard. It will take a while to build and once its done, you should be able to see chatbot section in the left side of the screen.


![alt text](screenshots/step17.png "testing the chatbot")

Go ahead and test it. 

## Conclusion

In this work shop we have build a chatbot powered by Lex, Lambda and HERE servies 

![alt text](screenshots/chatbot.png "testing the chatbot")