---
permalink: /network/
title: Network
keywords: network
summary: ""
---

# The EXP Network

- [Overview](#overview)
- [Channels](#channels)
  - [System Flag](#system-flag)
  - [Consumer Flag](#consumer-flag)
- [Broadcasting](#broadcasting)
- [Listening](#listening)
- [Responding](#responding)
- [Examples](#examples)

## Overview

**`exp.get_devices()`**

Does something interesting (test).

```javascript

const exp = require('exp')
exp.get_devices()

```

**`exp.get_asd()`**

The EXP network faciliates real time communication across the EXP platform. Messaging takes place in the form of [broadcasts](#broadcasting) and [responses](#responding). Broadcasts contain a name and JSON serializable payload and are emitted onto a specified [channel](#channels). Entities can listen on channels for broadcasts with a specified name, and respond to the broadcast with a JSON serializable payload. The broadcaster receives a JSON array of response payloads.


## Channels

Channels provide namespacing for messaging and help reduce network traffic by routing broadcasts to only devices that are actively listening on the given channel.

In addition to a channel name, channels have two flags `system` and `consumer`. The channel name can be any string. The full channel is defined by the combination of the organization, channel name, and these two flags. Channels with the same name, but different flags are **NOT** the same channel.


### System Flag

Broadcasts sent directly by EXP system will always be on `system` channels. Trying to emit a broadcast on a `system` channel will result in an error.

### Consumer Flag

As consumer app credentials are typically delivered with a mobile application to the consumer, messages from consumer apps should be treated with caution. For this reason, all broadcasts from consumer app devices can only be on emitted on `consumer` channels. Consumer apps can also only listen for messages on `consumer` channels. This allows non-consumer devices to communicate freely on non-consumer channels without having to worry about abuse of consumer app credentials.


### Advanced Topic: Generating a Channel ID

Under the hood, channels are actually defined by a hash of the organization, name, and flags. This is refered to as the channel ID. The hash can be generated by creating a JSON string from the array [org, name, system, consumer] where system and consumer are 0 or 1 (0 for false, 1 for true). Urlsafe Base 64 encoding the resulting string is the channel id.




## Broadcasting


```javascript
exp.getChannel('myChannel').broadcast('hello', {}, 500).then(responses => {});
```


```python
responses = exp.get_channel('my channel').broadcast('hello', None, 500)
```




Sending a broadcast is as simple as sending a HTTP POST to `/api/networks/current/broadcasts`.

```javascript
{
  "name": "myEventName",
  "payload": {
    "anything": 2,
    "anything2": ["hi!"]
  },
  "channel": {
    "name": "myChannelName",
    "consumer": true
  }
}
```

The broadcast is made up of 3 components, the channel, the message name, and the message payload. In this case the consumer flag is set.

The response to the HTTP request is a list of response payloads that were received while the HTTP request was open.

To specify the amount of wait for responses, add a `timeout` query parameter to the broadcast, specifying the number of milliseconds to wait for responses to the broadcast: i.e. `/api/networks/current/broadcasts?timeout=5000`. The HTTP request will be kept open for approximately the given timeout.




## Listening


### Advanced Topic: Manual Connection

Listening for broadcasts requires an active socket connection to the EXP Network. The easiest way to listen is to use one of the EXP SDKs.

The authentication field `network` contains information about which host you shoudl establish a connection to in order to listen for events. EXP network servers use the SocketIO protocol. Using a SocketIO client, connect to the host provided in `network.host` and pass your authentication token as the `token` query param. Once a connection is established, you will need to subscribe to a channel. Send the `subscribe` event with a list of channel ids you wish to listen on. You will subsequently receive a `subscribed` event containing a list of channel ids you are now subscribed to. If you get disconnected you will need to resubscribe to all channels.

Once subscribed to a channel, you will begin to receive broadcasts. The broadcast payload looks like:

```
{
  "id": ""
  "name": ""
  "channel": ""
  "payload": ""
}
```


# TEMP STUFF
# Examples


## Creating a Device and Listening for Updates

Updates to API resources are sent out over a system channel with the event name "update".

```javascript
exp.createDevice({ 'name': 'my_sweet_device' }).then(device => {
  return device.save().then(() => {
    device.getChannel({ system: true }).listen('update', payload => {
      console.log('Device was updated!');
    });
  });
});
```


## Modifying a Resource in Place

```javascript
exp.getExperience('[uuid]').then(experience => {
  experience.document.name = 'new name';
  return experience.save()
});
```


## Using The EXP Network

The EXP network facilitates real time communication between entities connected to EXP. A user or device can broadcast a JSON serializable payload to users and devices in your organization, and listeners to those broadcasts can respond to the broadcasters.

### Channels

All messages on the EXP network are sent over a channel. Channels have a name, and two flags: ```system``` and ```consumer```.

```javascript
const channel = exp.getChannel('myChannel', { system: false, consumer: false });
```

Use ```system: true``` to get a system channel. You cannot send messages on a system channels but can listen for system notifications, such as updates to API resources.

Use ```consumer: true``` to get a consumer channel. Consumer devices can only listen or broadcast on consumer channels. When ```consumer: false``` you will not receive consumer device broadcasts and consumer devices will not be able to hear your broadcasts.

Both ```system``` and ```consumer``` default to ```false```. Consumer devices will be unable to broadcast or listen to messages on non-consumer channels.


### Broadcasting

Use the broadcast method of a channel object to send a named message containing an optional JSON serializable payload to other entities on the EXP network. You can optionally include a timeout to wait for responses to the broadcast. The broadcast will return a promise and resolve approximately after the given timeout (milliseconds) with a ```list``` of response payloads. Each response payload can any JSON serializable type.

```javascript
exp.getChannel('myChannel').broadcast('myEvent', 'hello', 5000).then(responses => {
  responses.forEach(response => console.log(response));
});
```

### Listening

To listen for broadcasts, call the listen method of a channel object.  Pass in the name of the event you wish to listen for and a callback to handle the broadcast. The callback will receive the broadcast payload. Listen returns a promise that resolves with a listener object when listening on the network has actually started. Calling cancel on the listener object stops the callback from being executed.


```javascript

exp.getChannel('myChannel').listen('myEvent', payload => {
  console.log('Event Received!');
  console.log(payload);
}).then(listener => {
  // Remove the listener after 5 seconds.
  setTimeout(() => listener.cancel(), 5000);
});

```


### Responding

To respond to a broadcast, call the second argument of the listener callback with your response.

```javascript

exp.getChannel('my_channel').listen('my_event', (payload, respond) => {
  if (payload === 'hello!') respond('hi there!');
});

```



## Starting the SDK

The SDK is started by calling ```exp.start``` and specifying your credentials and configuration options. You may supply user, device, or consumer app credentials. You can also authenticate in pairing mode.

Users must specify their ```username```, ```password```, and ```organization```.

```javascript
exp.start({ username: 'joe@joemail.com', password: 'JoeRocks42', organization: 'joesorg' });
```

Devices must specify their ```uuid``` and ```secret``` .
```javascript
exp.start({ uuid: '[uuid]', secret: '[secret]' });
```

Consumer apps must specify their ```uuid``` and ```apiKey```.

```javascript
exp.start({ uuid: '[uuid]', apiKey: '[api key]');
```



## Advanced Options



# API Resources

# The EXP Network

# Advanced Topics

## Context Based Memory Management

A ```context``` is a string that can be used to track event listener registration. A copy of an instance of the SDK can be created by calling ```clone(context)``` method. This will return a cloned instance of the SDK bound to the given context. Calling ```clear(context)``` on any SDK instance derived from the original would remove all event listeners registered by the SDK instance bound to that context. This feature is generally intended for use by player apps, but it provides a simple interface to memory management of event listeners and can help reduce the chances of memory leaks due to orphaned event listener registrations in complex or long lived applications.

In this example, a module starts the SDK, and exports two cloned instances of the SDK bound to different contexts. After some interval, we clear all the event listeners bound to one of the context:

```javascript
const exp = require('exp-sdk');

exp.start(options);
exp1 = exp.clone('context 1')
exp2 = exp.clone('context 2');

exp1.getChannel('myChannelName').listen('myEventName', () => {});
exp2.getChannel('myChannelName').listen('myEventName', () => {});

exp2.clear(); // Unregisters all callbacks and listerers attached by exp2.

```

Note that calling ```stop``` on a cloned SDK instance or the original sdk instance stops the sdk for the original and all derived clones.


## Using Multiple Instances of the SDK

Calling ```exp.fork()``` will return an unstarted instance of the sdk.

```javascript

const exp = require('exp-sdk');
const exp2 = exp.fork();

exp.start(options1);
exp2.start(options2)

exp.stop();
exp2.stop();

```


Context based memory management is also supported when using multiple instances of the SDK, but unique names must be used for each context as event listeners are registered in a global pool.

```javascript
const exp = require('exp-sdk');
const sdk1 = exp.start(options1);
const sdk2 = exp.start(options2);
sdk1A = sdk1.clone('A');
sdk1B = sdk1.clone('B');
sdk2A = sdk2.clone('A');

sdk2.clear('A'); // Also clears sdk1A!

```

