# Botkit Controller Object

#### controller.hears()
| Argument | Description
|--- |---
| patterns | An _array_ or a _comma separated string_ containing a list of regular expressions to match
| types  | An _array_ or a _comma separated string_ of the message events in which to look for the patterns
| middleware function | _optional_ function to redefine how patterns are matched. see [Botkit Middleware](middleware.md)
| callback | callback function that receives a message object

Example:
```javascript
controller.hears(['keyword','^pattern$'],['message_received'],function(bot,message) {

  // do something to respond to message
  bot.reply(message,'You used a keyword!');

});
```

When using the built in regular expression matching, the results of the expression will be stored in the `message.match` field and will match the expected output of normal Javascript `string.match(/pattern/i)`.

For example:

```javascript
controller.hears('open the (.*) doors',['message_received'],function(bot,message) {
  var doorType = message.match[1]; //match[1] is the (.*) group. match[0] is the entire group (open the (.*) doors).
  if (doorType === 'pod bay') {
    return bot.reply(message, 'I\'m sorry, Dave. I\'m afraid I can\'t do that.');
  }
  return bot.reply(message, 'Okay');
});
```


#### controller.on()
| Argument | Description
|--- |---
| event_name | a string or array containing an event or comma-separated list of events
| callback  | callback function in the form of form of function(bot, event) {...}

Handle events emitted by Botkit. The vast majority of events will call a callback function with 2 parameters - a bot instance, and the event object.  

[Read about receiving and handling events](core.md#responding-to-events)

```javascript
// handle a message event
controller.on('message_received', function(bot, message) {
  bot.reply(message,'Received');
});

// handle a channel join event
controller.on('channel_join', function(bot, event) {
  bot.reply(event,'Welcome to the channel!');
});
```

Note that you may also trigger your own events using [controller.trigger()](#controllertrigger) and handle them. This can be
useful for separating the trigger logic from the actual event handlers, as in the example below.

```javascript

// listen for a help request.. and then trigger a help_request event.
controller.hears('help', 'message_received', function(bot, message) {
  // this event can be triggered whenever a user needs help
  bot.trigger('help_request', [bot, message]);
});

controller.on('help_request', function(bot, message) {

  bot.reply(message,'I am here to help!');

});
```

#### controller.trigger()
| Argument | Description
|--- |---
| event_name | the name of a custom event
| parameters | an array of parameters to be passed on to any handler functions

Triggers a custom event, which can then be handled by [controller.on()](#controlleron).

The second argument to `controller.trigger()` is an array which should contain an array of parameters that will be passed on to any handler functions. While not required, it is highly recommended that custom events conform to the pattern used by native Botkit events and include a bot instance as the first parameter, and if applicable, a normalized Botkit event object as the second paramter:

```javascript
controller.trigger('my_custom_event', [bot, event]);
```

#### controller.changeEars()
| Argument | Description
|--- |---
| test_function | a function in the form function(tests_array, message_object)

This function can be used to modify the way Botkit's hearing system works by replacing the pattern matching function with a custom function.

The test function must perform synchronous tests, and should return true if the message represents a match, and otherwise return false.

Code based on Botkit's built-in regular expression test is below:

```
var hears_regexp = function(tests, message) {
    for (var t = 0; t < tests.length; t++) {
        if (message.text) {

            // the pattern might be a string to match (including regular expression syntax)
            // or it might be a prebuilt regular expression
            var test = null;
            if (typeof(tests[t]) == 'string') {
                try {
                    test = new RegExp(tests[t], 'i');
                } catch (err) {
                    botkit.log('Error in regular expression: ' + tests[t] + ': ' + err);
                    return false;
                }
                if (!test) {
                    return false;
                }
            } else {
                test = tests[t];
            }

            if (match = message.text.match(test)) {
                message.match = match;
                return true;
            }
        }
    }
    return false;
};

controller.changeEars(hears_regexp);
```

#### controller.excludeFromConversations()

| Argument | Description
|--- |---
| events | single event or an array of events to exclude from conversations

Messaging platforms send a dizzying array of event types - and new ones are added all the time!

Sometimes, it is desirable to exclude certain event types from inclusion in Botkit conversations. To do this, call `excludeFromConversations()` after creating your controller object.

For example:

```javascript
// always exclude facebook postback events from conversations
controller.excludeFromConversations('facebook_postback')
```

#### controller.spawn()
| Argument | Description
|--- |---
| configuration | an object with instance-specific configuration options
| callback | a function that will receive the new bot instance as a parameter

Spawn a new instance of the bot in order to send messages or handle replies.

Whenever the bot is doing something in response to an incoming message, Botkit will handle spawning bot instances for you.  However, in the event that your bot needs to send an alert or subscription message that is not directly in reply to an incoming message, spawning a bot is required.

The configuration options required to spawn a bot instance differ from platform to platform. In many cases, no additional configuration is required. Refer to the platform specific documnentation for more detail.

```javascript
function sendAlertToBot(alert_message) {

  controller.spawn({}, function(bot) {

    bot.say(alert_message);

  });

}
```

#### controller.defineBot()
| Argument | Description
|--- |---
| bot_constructor | function of the form function(botkit_controller, configuration) which returns a bot instance

This function is used to create new platform connectors for Botkit, and is used to define the specific behaviors and features of that platform's bot instance.

For more information, [read the guide to creating new platform connectors for Botkit](howto/build_connector.md)


#### controller.setTickDelay()
| Argument | Description
|--- |---
| delay | time between event loops

Adjust the speed of Botkit's event loops. By default, this is set to 1500ms, which means that Botkit will evaluate all active conversations and send any pending replies every 1.5 seconds.

Decreasing this time will increase the speed at which bots send replies in conversations.

For example:
``` javascript
// only wait 100ms between conversation loops
controller.setTickDelay(100);
```

#### controller.startTicking()

This function begins Botkit's internal event loop, which will tick at an interval of once every 1.5 seconds unless modified using [controller.setTickDelay()](#controller-settickdelay)

In most cases, this is called internally by a platform connector and does not have to be called directly by bot code.

However, if your bot seems to freeze up after the first message in any conversation, you may need to call it!


#### controller.setupWebserver(port, cb)
| Argument | Description
|--- |---
| port | a port number for the webserver
| callback | a function in the form function(err, webserver) {...}

Create an instance of the [Express.js webserver](https://expressjs.com) for use with platform-specific features like oauth and incoming webhooks.

Botkit provides a simple Express webserver for those who don't know or don't want to set up their own. However, if you want to do anything with your webserver other than those required by Botkit, it is our recommendation that you manage your own instance of Express. A good place to see this in action is [inside all of the botkit starter kits](starterkits.md).

#### controller.userAgent()

Returns a user agent string for use when making API requests to external resources.

#### controller.version()

Returns the current package version of Botkit's core library

#### controller.shutdown()

This function _stops_ the event loop from processing active conversations. It is the opposite of [controller.startTicking()](#controller-startticking)
