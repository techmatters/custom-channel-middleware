# Customer Channel Middleware

Middleware for custom channels to feed into [Twilio Flex](https://www.twilio.com/flex)

## Twitter

The following guide demonstrates how you can enable Twilio Flex to communicate via Twitter DMs.

### Setup a Twilio Flex instance and middleware

Follow [this guide](https://www.twilio.com/blog/add-custom-chat-channel-twilio-flex) to setup your Flex instance, including the optional portions.

### Obtain a Twitter developer account

You will need access to the Twitter API, which you can apply for [here](https://developer.twitter.com/en/apply-for-access).

Setup an App and update the permissions to `Read, Write, and Direct Messages`. Note that you will need to regenerate your credentials when you modify this setting.

Create a developer environment [here](https://developer.twitter.com/en/account/environments) for the `Account Activity API` and name it `dev`.

### Install autohook

Once you have your Twitter developer account, you can set up the environment variables as described [here](https://github.com/twitterdev/autohook#dotenv-envtwitter). Installation instructions can be found [here](https://github.com/twitterdev/autohook#install).

### Setup Twilio serverless function

The Twilio serverless function will be responsible for responding to Twitter DMs via Twilio Flex.

After you create the function, replace the code with the following:

```javascript
exports.handler = function(context, event, callback) {
  if (event['Source'] == 'SDK') {
        let Twit = require('twit')
        let T = new Twit({
            consumer_key:         process.env.TWITTER_CONSUMER_KEY,
            consumer_secret:      process.env.TWITTER_CONSUMER_SECRET,
            access_token:         process.env.TWITTER_ACCESS_TOKEN,
            access_token_secret:  process.env.TWITTER_ACCESS_TOKEN_SECRET,
            timeout_ms:           60*1000,  // optional HTTP request timeout to apply to all requests.
            strictSSL:            true,     // optional - requires SSL certificates to be valid.
        })
        let sendTo = {
            event: {
                type: 'message_create',
                message_create: {
                    target: {
                        recipient_id: event['recipient_id'],
                    },
                    message_data: {
                        text: event['Body'],
                    },
                },
            }
        }
        T.post("direct_messages/events/new", sendTo, function(err, data, response){
                console.info(data);
                });
        };
  }
```

In the Settings section, add the following dependency: `twit` with version `2.2.11`.

In the Settings section, add the following environment variables: `TWITTER_ACCESS_TOKEN_SECRET`, `TWITTER_ACCESS_TOKEN`, `TWITTER_CONSUMER_SECRET`, `TWITTER_CONSUMER_KEY`  

Now, save your function and click `Deploy All` and copy the URL for your function, you will need it to update the middleware.

### Update middleware

In `./twilio-flex-custom-webchat/middleware/server.js`, replace all the code there with:

```javascript
require('dotenv').load();

const flex = require('./flex-custom-webchat');

const { Autohook } = require('twitter-autohook');
(async ƛ => {
  const webhook = new Autohook();
  // Removes existing webhooks
  await webhook.removeWebhooks();
  // Listen for incoming direct messages
  webhook.on('event', async event => {
    if (event.direct_message_events) {
      let active_user = event.for_user_id;
      let sender_id = event.direct_message_events[0].message_create.sender_id
      if(sender_id != active_user){
        console.log("New message from: " + event.users[sender_id].name)
        console.log(event.direct_message_events[0].message_create.message_data.text)
        await flex.sendMessageToFlex(event.direct_message_events[0].message_create.message_data.text, event.users[sender_id].name, event.users[sender_id].screen_name, event['for_user_id']);
      }
    }
  });
  // Starts a server and adds a new webhook
  await webhook.start();
  // Subscribes to a user's activity
  await webhook.subscribe({oauth_token: process.env.TWITTER_ACCESS_TOKEN, oauth_token_secret: process.env.TWITTER_ACCESS_TOKEN_SECRET});
})();
```

In `./twilio-flex-custom-webchat/middleware/flex-custom-webchat.js`, replace the code with:

```javascript
require('dotenv').config();
const fetch = require('node-fetch');
const { URLSearchParams } = require('url');
var base64 = require('base-64');

const client = require('twilio')(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

var flexChannelCreated;

function sendChatMessage(serviceSid, channelSid, chatUserName, recipientId, body) {
  console.log('Sending new chat message');
  const params = new URLSearchParams();
  params.append('RecipientId', recipientId);
  params.append('Body', body);
  params.append('From', chatUserName);
  console.log('Params:' + params)
  return fetch(
    `https://chat.twilio.com/v2/Services/${serviceSid}/Channels/${channelSid}/Messages`,
    {
      method: 'post',
      body: params,
      headers: {
        'X-Twilio-Webhook-Enabled': 'true',
        Authorization: `Basic ${base64.encode(
          `${process.env.TWILIO_ACCOUNT_SID}:${process.env.TWILIO_AUTH_TOKEN}`
        )}`
      }
    }
  );
}

function createNewChannel(flexFlowSid, flexChatService, recipientId, chatUserName) {
  return client.flexApi.channel
    .create({
      flexFlowSid: flexFlowSid,
      identity: recipientId,
      chatUserFriendlyName: chatUserName,
      chatFriendlyName: 'Flex Custom Chat',
      target: chatUserName
    })
    .then(channel => {
      console.log(`Created new channel ${channel.sid}`);
      return client.chat
        .services(flexChatService)
        .channels(channel.sid)
        .webhooks.create({
          type: 'webhook',
          'configuration.method': 'POST',
          'configuration.url': `<Twilo Flex function URL>?channel=${channel.sid}&recipient_id=${recipientId}`,
          'configuration.filters': ['onMessageSent']
        })
        .then(() => client.chat
        .services(flexChatService)
        .channels(channel.sid)
        .webhooks.create({
          type: 'webhook',
          'configuration.method': 'POST',
          'configuration.url': `<Twilo Flex function URL>/channel-update`,
          'configuration.filters': ['onChannelUpdated']
        }))
    })
    .then(webhook => webhook.channelSid)
    .catch(error => {
      console.log(error);
    });
}

async function resetChannel(status) {
  if (status == 'INACTIVE') {
    flexChannelCreated = false;
  }
}

async function sendMessageToFlex(msg, user, screen_name, sender_id) {
  flexChannelCreated = await createNewChannel(
      process.env.FLEX_FLOW_SID,
      process.env.FLEX_CHAT_SERVICE,
      sender_id,
      user
  );
  console.log(flexChannelCreated)
  console.log('DM from: ' + screen_name)
  sendChatMessage(
      process.env.FLEX_CHAT_SERVICE,
      flexChannelCreated,
      user,
      sender_id,
      msg
  );
}

exports.sendMessageToFlex = sendMessageToFlex;
exports.resetChannel = resetChannel;
```

Replace `<Twilo Flex function URL>` with the URL obtained from your Twilio Function.

Launch the middleware in the directory `<path-to>/twilio-flex-custom-webchat/middleware` with the command `node server.js`.

### Try it out!

[Login](https://flex.twilio.com) to your Flex instance and set your agent's status to be `Available`.

DM the Twitter account associated with the Twitter developer account and [navigate to your agent desktop](https://flex.twilio.com/agent-desktop).

Accept the message and respond.

Celebrate!

#### References
* https://www.twilio.com/blog/add-custom-chat-channel-twilio-flex
* https://github.com/vernig/twilio-flex-custom-webchat
* https://github.com/twitterdev/autohook
* https://github.com/ttezel/twit

#### Contributors
* Nick Hurlburt
* Ashkon Honardoost
* Deepak Srikanth
* Elmer Thomas
* Toby Allen
