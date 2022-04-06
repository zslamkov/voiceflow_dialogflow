# Voiceflow x Dialogflow ES
## Prerequisites 

Here are the tools you will need for this project:

1. Dialogflow ES Agent
2. Voiceflow Account

## Voiceflow authentication
For authentication, we will need to get our VF Project API key. 

To access the Project API key for a specific project:

1. Open the project you want to connect with
2. Select on the Integrations tab (shortcut: 3)
3. Copy the Dialog API Key.

![project api](https://user-images.githubusercontent.com/68556615/161978440-7c6a2605-5721-489e-ae1b-a9fd68db84e7.png)


Add the credentials into your `.env` file
```
VF_API_KEY= "VF.xxxxx"
```

## Setting up the Project

Install and run the project:

1. Clone this repo:
```bash
git clone https://github.com/zslamkov/voiceflow_dialogflow.git
```

2. Install dependencies:
```bash
npm install
```

## DialogFlow API

To setup the Dialogflow API with NodeJS, add the following code and add your credentials . To retrieve your credentials, see this [video](https://youtu.be/dFN79tEr_bc)

```js
const dialogflow = require("@google-cloud/dialogflow");

const CREDENTIALS = JSON.parse(process.env.CREDENTIALS);
const PROJECTID = CREDENTIALS.project_id;

const CONFIGURATION = {
  credentials: {
    private_key: CREDENTIALS["private_key"],
    client_email: CREDENTIALS["client_email"],
  },
};

const sessionClient = new dialogflow.SessionsClient(CONFIGURATION);
```

## Detect intent
The below `detectIntent` function is responsible for sending a user utterance to our Dialogflow agent and responding with the triggered intent and any relevant entity values.

Other data that may be useful in the `detectIntent` response body include languageCode. For more information of what is available, see [here](https://cloud.google.com/dialogflow/es/docs/reference/rest/v2/DetectIntentResponse).

```js
const detectIntent = async (languageCode, queryText, sessionId) => {
  let sessionPath = sessionClient.projectAgentSessionPath(PROJECTID, sessionId);

  let request = {
    session: sessionPath,
    queryInput: {
      text: {
        text: queryText,
        languageCode: languageCode,
      },
    },
  };
  const responses = await sessionClient.detectIntent(request);
  const result = responses[0].queryResult;

  const confidence = result.intentDetectionConfidence;

  const obj = result.parameters.fields;
  const createIntent = (name, value) => ({ name, value });
  const intents = Object.entries(obj).map(([key, value]) =>
    createIntent(key, value.stringValue)
  );

  return {
    response: result.intent.displayName,
    entities: intents,
    confidence: confidence,
  };
};
```


## NodeJS App

The below app allows us to interact with our chat assistant via the CLI. 

We will be retrieving the `userID` by having the user enter their name. Once completed, we will send a launch request to Voiceflow to start the conversation and send the first steps to the channel. 


```js
async function main() {
  const userID = await cli.prompt("> What is your name?");
  // send a simple launch request starting the dialog
  let isRunning = await interact(userID, { type: "launch" });
```
Now on each new interaction from the user, we will be passing the userID and text input through the `detectIntent` function and returning the `intent` object that will be used in the next line of code to populate the request payload to Voiceflow. 

```js
  while (isRunning) {
    const nextInput = await cli.prompt("> Say something");
    // send a simple text type request with the user input
    let intent = await detectIntent(nextInput, userID);
    
    isRunning = await interact(userID, {
      type: "intent",
      payload: {
        intent: {
          name: intent.response,
        },
        entities: intent.entities,
        confidence: intent.confidence,
      },
    });
  }
  console.log("The end! Start me again with `npm start`");
}

main();
```

## Run
What it might look like in action:

```
$ npm start

> What is your name?: zoran
what can I do for you?
...
> Say something: send email
who is the recipient?
...
> Say something: zoran@voiceflow.com
what is the title of your email?
...
> Say something: How was your day?
sending the email for zoran@voiceflow.com called "How was your day?". Is that correct?
...
> Say something: yes
successfully sent the email for zoran@voiceflow.com called "How was your day?"
The end! Start me again with `npm start`
```
