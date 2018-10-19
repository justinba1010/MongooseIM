### Module Description

This module is a backend of [mod_event_pusher] that enables support for the
RabbitMQ integration. Currently there are 5 available notifications:

* **user presence changed** - Carries the user id (full jid by default) and
a boolean field corresponding to the current user online status.
* **private message sent/received** - Carries the user ids (both sender and
receiver) along with the message body.
* **group message sent/received** - Carries the user id and the room id
(full jids by default) along with the message body.

All these notifications are sent as JSON strings to RabbitMQ topic exchanges.
Each type of the notifications is sent to it's dedicated exchange. There are
three exchanges created on startup of the module, for presences, private
messages and group chat messages related events.

Messages are published to a RabbitMQ server with routing key being set to a user
bare jid (`user@domain`) and configurable topic e.g `alice@localhost.private_message_sent`.

### Options

* **amqp_host** (charlist, default: `"localhost"`) - Defines RabbitMQ server host (domain or IP address);
* **amqp_port** (integer, default: `5672`) - Defines RabbitMQ server AMQP port;
* **amqp_username** (string, default: `<<"guest">>`) - Defines RabbitMQ server username;
* **amqp_password** (string, default: `<<"guest">>`) - Defines RabbitMQ server password;
* **presence_exchange** (string, default: `<<"presence">>`) - Defines RabbitMQ presence exchange name;
* **chat_msg_exchange** (string, default: `<<"chat_msg">>`) - Defines RabbitMQ chat message exchange name;
* **groupchat_msg_exchange** (string, default: `<<"groupchat_msg">>`) - Defines RabbitMQ group chat message exchange name;
* **chat_msg_sent_topic** (string, default: `<<"chat_msg_sent">>`) - Defines RabbitMQ chat message sent topic name;
* **chat_msg_recv_topic** (string, default: `<<"chat_msg_recv">>`) - Defines RabbitMQ chat message received topic name;
* **groupchat_msg_sent_topic** (string, default: `<<"groupchat_msg_sent">>`) - Defines RabbitMQ group chat message sent topic name;
* **groupchat_msg_recv_topic** (string, default: `<<"groupchat_msg_recv">>`) - Defines RabbitMQ group chat message received topic name;
* **pool_size** (integer, default: `100`) - Worker pool size for publishing notifications.

### Example configuration

```Erlang
{mod_event_pusher, [
    {backends, [
        {rabbbit, [
            {amqp_host, "localhost"},
            {amqp_port, 5672},
            {amqp_username, <<"guest">>},
            {amqp_password, <<"guest">>},
            {presence_exchange, <<"presence">>},
            {chat_msg_exchange, <<"chat_msg">>},
            {chat_msg_sent_topic, <<"chat_msg_sent">>},
            {chat_msg_recv_topic, <<"chat_msg_recv">>},
            {groupchat_msg_exchange, <<"groupchat_msg">>},
            {groupchat_msg_sent_topic, <<"groupchat_msg_sent">>},
            {groupchat_msg_recv_topic, <<"groupchat_msg_recv"},
            {pool_size, 50}
        ]}
    ]}
]}
```

### JSON Schema examples
The different kinds of notifications deliver slightly different messages.
The messages are delivered in a JSON format.
#### Presence updates

The JSON format for an online presence update notification is:
```JSON
{
    "user_id": "alice@localhost/res1",
    "present": true
}
```

For offline presence updates, the `present` boolean value is set to false:

```JSON
{
    "user_id": "alice@localhost/res1",
    "present": false
}
```
#### Sent/received messages
The JSON format for a private message notification is:
```JSON
{
    "to_user_id": "bob@localhost/res1",
    "message": "Hello, Bob",
    "from_user_id": "alice@localhost/res1"
}
```
The notification is similar for group messages. For example for "sent" events:
```JSON
{
    "to_user_id": "muc_publish@muc.localhost",
    "message": "Hi, Everyone!",
    "from_user_id": "bob@localhost/res1"
}
```
and for "received" events:

```JSON
{
    "to_user_id": "bob@localhost/res1",
    "message": "Hi, Everyone!",
    "from_user_id": "muc_publish@muc.localhost/alice"
}
```

### Metrics

The module provides some metrics related to RabbitMQ connections and messages
as well. Provided metrics:

  * `connections_active` - number of active connections to a RabbitMQ
  server
  * `connections_opened` - number of opened connections to a RabbitMQ
  server since module startup
  * `connections_closed` - number of closed connections to a RabbitMQ
  server since module startup
  * `connections_failed` - number of failed connections to a RabbitMQ
  server since module startup
  * `messages_published` - number of published messages to a RabbitMQ server
  since module startup
  * `messages_timeout` - number of messages to a RabbitMQ server that weren't
  confirmed by the server since module startup
  * `messages_failed` - number of messages to a RabbitMQ server that were
  rejected by the server since module startup
  * `message_publish_time` - amount of time it takes to publish a message to
  a RabbitMQ server and receives a confirmation
  * `message_payload_size` - size of a messages (in bytes) that was published to
  a RabbitMQ server

> All the above metrics have a prefix which looks as follows:  
> `mongooseim.<xmpp_host>.backends.mod_event_pusher_rabbit.<metric_name>`.
> For example a proper metric name would look like:
> `mongooseim.localhost.backends.mod_event_pusher_rabbit.connections_active`

### Current status

This module is still in an experimental phase.

### Guarantees

There are no guarantees. The current implementation uses "best effort" approach
which means that we don't care if a message is delivered to a RabbitMQ server.
If a message couldn't be delivered to the server for any reason the module
just updates appropriate metrics and print some log messages.

### Publisher confirms

One-to-one confirmations are in use. When a worker sends a message to a RabbitMQ
server it waits for a confirmation from the server before it starts to process
next message. This approach allows to introduce backpressure on a RabbitMQ server
connection cause the server can reject messages when it's overloaded. On the
other hand it can cause performance degratation.

### Worker selection strategy

The module uses `wpool` library for managing worker processes  and `avaiable_worker`
strategy is in use. Different strategies imply different behaviours of the system.

#### Event messages queueing

When `available_worker` strategy is set all the event messages are queued in
single worker pool manager process state. When diffrenet strategy is set e.g
`next_worker` those messages are placed in worker processes inboxes. There is no
possiblity to change the worker strategy for now.

#### Event messages ordering

`available_worker` strategy does not ensure that user events will be delivered to
a RabbitMQ server properly ordered in time.

[mod_event_pusher]: ./mod_event_pusher.md