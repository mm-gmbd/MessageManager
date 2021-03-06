# MessageManager Library 0.0.2 (Alpha)

MessageManager is framework for asynchronous bidirectional agent to device communication. 
The library is a successor of [Bullwinkle](https://github.com/electricimp/Bullwinkle).

The work on MessageManager was inspired by complains on the good old 
[Bullwinkle](https://github.com/electricimp/Bullwinkle), which
did not prove itself to be the most optimized library from the footprint standpoint 
(timers, dynamic memory and traffic utilization). 

So we started working on a completely new library to address these concerns which ended up to be
[MessageManager](https://github.com/electricimp/MessageManager).

## Some MessageManager Features

- Optimized system timers utilization
- Optimized traffic used for service messages (replies and acknowledgements)
- Connection awareness (leveraging optional 
[ConnectionManager](https://github.com/electricimp/ConnectionManager) library and 
device/agent.send status)
- Separation of the message payload from the wrapping structure that may be used and 
extended for application specific purposes as it's not sent over the wire
- Per-message timeouts
- API hooks to manage outgoing messages. This may be used to adjust application 
specific identifiers or timestamps, introduce additional fields and meta-information or to enqueue message 
to delay delivery
- API hooks to control retry process. This may be used to dispose outdated messages or to change it's meta-information

## API Overview
- [MessageManager](#mmanager) - The core library - used to add/remove handlers, and send messages
    - [MessageManager.send](#mmanager_send) - Sends the data message
    - [MessageManager.beforeSend](#mmanager_before_send) - Sets the callback which will be called before a message is sent
    - [MessageManager.beforeRetry](#mmanager_before_retry) - Sets the callback which will be called before a message is retried
    - [MessageManager.on](#mmanager_on) - Sets the callback, which will be called when a message with the specified name is received
    - [MessageManager.onFail](#mmanager_on_fail) - Sets the handler to be called when an error occurs
    - [MessageManager.onAck](#mmanager_on_ack) - Sets the handler to be called on the message acknowledgement
    - [MessageManager.onReply](#mmanager_on_reply) - Sets the handler to be called when the message is replied
    - [MessageManager.getPendingCount](#mmanager_get_pending_count) - Returns the overall number of pending messages 
    (either waiting for acknowledgement or hanging in the retry queue)

### Details and Usage

<div id="mmanager"><h4>Constructor: MessageManager(<i>[options,] [cm]</i>)</h4></div>

Calling the MessageManager constructor creates a new MessageManager instance. An optional *options* 
table can be passed into the constructor to override default behaviours.

<div id="mmanager_options"><h5>options</h5></div>
A table containing any of the following keys may be passed into the MessageManager constructor to modify the default behavior:

| Key | Data Type | Default Value | Description |
| ----- | -------------- | ------------------ | --------------- |
| *debug* | Boolean | `false` | The flag that enables debug library mode, which turns on extended logging. |
| *retryInterval* | Integer | 10 | Changes the default timeout parameter passed to the [retry](#mmanager_retry) method. |
| *messageTimeout* | Integer | 10 | Changes the default timeout required before a message is considered failed (to be acknowledged or replied). |
| *autoRetry* | Boolean | `false` | If set to `true`, MessageManager will automatically continue to retry sending a message until *maxAutoRetries* has been reached when no [onFail](#mmanager_on_fail) handler is supplied. Please note if *maxAutoRetries* is set to 0, *autoRetry* will have no limit to the number of times it will retry. |
| *maxAutoRetries* | Integer | 0 | Changes the default number of automatic retries to be peformed by the library. After this number is reached the message will be dropped. Please not the message will automatically be retried if there is when no [onFail](#mmanager_on_fail) handler registered by the user. |

<div id="mmanager_connection_manager"><h5>cm</h5></div>

Optional instance of [ConnectionManager](https://github.com/electricimp/ConnectionManager) library that helps MessageManager
to track the connectivity status.

##### Examples

```squirrel
// Initialize using default settings
local mm = MessageManager();
```

```squirrel
// MessageManager options
local options = {
    "debug": true,
    "retryInterval": 15,
    "messageTimeout": 2,
    "autoRetry": true,
    "maxAutoRetries": 10
}

// Create ConnectionManager instance
local cm = ConnectionManager({
    "blinkupBehavior": ConnectionManager.BLINK_ALWAYS,
    "stayConnected": true
});
imp.setsendbuffersize(8096);

local mm = MessageManager(options, cm);
```

<div id="mmanager_send"><h4>send(<i>name[, data, timeout]</i>)</h4></div>

Sends a named message to the partner’s MessageManager instance. 

The optional *data* parameter can be a basic 
Squirrel type (`1`, `true`, `"A String"`) or more complex data structures such as an array or table, 
but it must be [a serializable Squirrel value](https://electricimp.com/docs/resources/serialisablesquirrel/).

The optional *timeout* parameter overrides the global `messageTimeout` setting per message.
 
```squirrel
mm.send("turnOn", true);
```

... TBC ...

### Other Usage Examples

#### Integration with [ConnectionManager](https://github.com/electricimp/ConnectionManager)

```squirrel
// Device code

#require "ConnectionManager.class.nut:1.0.2"
#require "MessageManager.class.nut:0.0.2"

local cm = ConnectionManager({
    "blinkupBehavior": ConnectionManager.BLINK_ALWAYS,
    "stayConnected": true
})

// Set the recommended buffer size 
// (see https://github.com/electricimp/ConnectionManager for details)
imp.setsendbuffersize(8096)

local config = {
    "msgTimeout": 2
}

local counter = 0
local mm = MessageManager(config, cm)

mm.onFail(
    function(msg, error, retry) {
        server.log("Error occurred: " + error)
        retry()
    }
)

mm.onReply(
    function(msg, response) {
        server.log("Response for " + msg.payload.data + " received: " + response)
    }
)

function sendData() {
    mm.send("name", counter++);
    imp.wakeup(1, sendData)
}

sendData()
```

```squirrel
// Agent code

#require "MessageManager.class.nut:0.0.2"

local mm = MessageManager()

mm.on("name", function(data, reply) {
    server.log("message received: " + data)
    reply("Got it!")
})
```

## License

Bullwinkle is licensed under the [MIT License](./LICENSE).
