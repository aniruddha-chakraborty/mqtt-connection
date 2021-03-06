mqtt-connection&nbsp;&nbsp;&nbsp;[![Build Status](https://travis-ci.org/mqttjs/mqtt-connection.png)](https://travis-ci.org/mqttjs/mqtt-connection)
===============

Barebone Connection object for MQTT.
Works over any kind of binary Streams, TCP, TLS, WebSocket, ...

[![JavaScript Style Guide](https://cdn.rawgit.com/feross/standard/master/badge.svg)](https://github.com/feross/standard)

It uses [mqtt-packet](http://npm.im/mqtt-packet) for generating and
parsing MQTT packets. See it for the full documentations on the
packet types.

  * <a href="#installation">Installation</a>
  * <a href="#usage">Usage</a>
  * <a href="#api">API</a>
  * <a href="#contributing">Contributing</a>
  * <a href="#license">License &amp; copyright</a>

This library is tested with node v4, v6 and v7. The last version to support
older versions of node was mqtt-connection@2.1.1.

Installation
-------

```sh
npm install mqtt-connection --save
```

Usage
-----

As a client:

```js
var net = require('net')
var mqttCon = require('mqtt-connection')
var stream = net.createConnection(1883, 'localhost')
var conn = mqttCon(stream)

// conn is your MQTT connection!
```

As a server:
```js
var net = require('net')
var mqttCon = require('mqtt-connection')
var server = new net.Server()

server.on('connection', function (stream) {
  var client = mqttCon(stream)

  // client connected
  client.on('connect', function (packet) {
    // acknowledge the connect packet
    client.connack({ returnCode: 0 });
  })

  // client published
  client.on('publish', function (packet) {
    // send a puback with messageId (for QoS > 0)
    client.puback({ messageId: packet.messageId })
  })

  // client pinged
  client.on('pingreq', function () {
    // send a pingresp
    client.pingresp()
  });

  // client subscribed
  client.on('subscribe', function (packet) {
    // send a suback with messageId and granted QoS level
    client.suback({ granted: [packet.qos], messageId: packet.messageId })
  })

  // timeout idle streams after 5 minutes
  stream.setTimeout(1000 * 60 * 5)

  // connection error handling
  client.on('close', function () { client.destroy() })
  client.on('error', function () { client.destroy() })
  client.on('disconnect', function () { client.destroy() })

  // stream timeout
  stream.on('timeout', function () { client.destroy(); })
})

// listen on port 1883
server.listen(1883)
```

As a websocket server:

```js
var websocket = require('websocket-stream')
var WebSocketServer = require('ws').Server
var Connection = require('mqtt-connection')
var http = require('http)
var server = http.createServer()

var wss = new WebSocketServer({server: server})

if (handler) {
  server.on('client', handler)
}

wss.on('connection', function (ws) {
  var stream = websocket(ws)
  var connection = new Connection(stream)

  handle(connection)
})

function handle (conn) {
  // handle the MQTT connection like
  // the net example
}
```

API
---

  * <a href="#connection"><code>mqtt.<b>Connection()</b></code></a>
  * <a href="#parseStream"><code>mqtt.<b>parseStream()</b></code></a>
  * <a href="#generateStream"><code>mqtt.<b>generateStream()</b></code></a>

---------------------------------

<a name="connection"></a>
### new mqtt.Connection([options])

Creates a new MQTT `Connection`.

Options:

  * `notData`: do not listen to the `'data'` event, so that it can
    respect backpressure. Pipe the `Connection` to another stream to
    consume the packets. If this option is passed `true` the object will
    emit no packet-related events.

#### Connection#connect(options, [callback])

Send a MQTT connect packet.

`options` supports the following properties:

* `protocolId`: Protocol ID, usually `MQIsdp`. `string`
* `protocolVersion`: Protocol version, usually 3. `number`
* `keepalive`: keepalive period in seconds. `number`
* `clientId`: client ID. `string`
* `will`: the client's will message options.
`object` that supports the following properties:
  * `topic`: the will topic. `string`
  * `payload`: the will payload. `string`
  * `qos`: will qos level. `number`
  * `retain`: will retain flag. `boolean`
* `clean`: the 'clean start' flag. `boolean`
* `username`: username for protocol v3.1. `string`
* `password`: password for protocol v3.1. `string`

#### Connection#connack(options, [callback])
Send a MQTT connack packet.

`options` supports the following properties:

* `returnCode`: the return code of the connack, success is
indicated by `0`. `number`

#### Connection#publish(options, [callback])
Send a MQTT publish packet.

`options` supports the following properties:

* `topic`: the topic to publish to. `string`
* `payload`: the payload to publish, defaults to an empty buffer.
`string` or `buffer`
* `qos`: the quality of service level to publish on. `number`
* `messageId`: the message ID of the packet,
required if qos > 0. `number`
* `retain`: retain flag. `boolean`

#### Connection#puback #pubrec #pubcomp #unsuback(options, [callback])
Send a MQTT `[puback, pubrec, pubcomp, unsuback]` packet.

`options` supports the following properties:

* `messageId`: the ID of the packet

#### Connection#pubrel(options, [callback])
Send a MQTT pubrel packet.

`options` supports the following properties:

* `dup`: duplicate message flag
* `messageId`: the ID of the packet

#### Connection#subscribe(options, [callback])
Send a MQTT subscribe packet.

`options` supports the following properties:

* `dup`: duplicate message flag
* `messageId`: the ID of the packet
* `subscriptions`: a list of subscriptions of the form
`[{topic: a, qos: 0}, {topic: b, qos: 1}]`

#### Connection#suback(options, [callback])
Send a MQTT suback packet.

`options` supports the following properties:

* `granted`: a vector of granted QoS levels,
of the form `[0, 1, 2]`
* `messageId`: the ID of the packet

#### Connection#unsubscribe(options, [callback])
Send a MQTT unsubscribe packet.

`options` supports the following properties:

* `messageId`: the ID of the packet
* `dup`: duplicate message flag
* `unsubscriptions`: a list of topics to unsubscribe from,
of the form `["topic1", "topic2"]`

#### Connection#pingreq #pingresp #disconnect(options, [callback])
Send a MQTT `[pingreq, pingresp, disconnect]` packet.

#### Event: 'connect'
`function(packet) {}`

Emitted when a MQTT connect packet is received by the client.

`packet` is an object that may have the following properties:

* `version`: the protocol version string
* `versionNum`: the protocol version number
* `keepalive`: the client's keepalive period
* `clientId`: the client's ID
* `will`: an object with the following keys:
  * `topic`: the client's will topic
  * `payload`: the will message
  * `retain`: will retain flag
  * `qos`: will qos level
* `clean`: clean start flag
* `username`: v3.1 username
* `password`: v3.1 password

#### Event: 'connack'
`function(packet) {}`

Emitted when a MQTT connack packet is received by the client.

`packet` is an object that may have the following properties:

* `returnCode`: the return code of the connack packet

#### Event: 'publish'
`function(packet) {}`

Emitted when a MQTT publish packet is received by the client.

`packet` is an object that may have the following properties:

* `topic`: the topic the message is published on
* `payload`: the payload of the message
* `messageId`: the ID of the packet
* `qos`: the QoS level to publish at

#### Events: \<'puback', 'pubrec', 'pubrel', 'pubcomp', 'unsuback'\>
`function(packet) {}`

Emitted when a MQTT `[puback, pubrec, pubrel, pubcomp, unsuback]`
packet is received by the client.

`packet` is an object that may contain the property:

* `messageId`: the ID of the packet

#### Event: 'subscribe'
`function(packet) {}`

Emitted when a MQTT subscribe packet is received.

`packet` is an object that may contain the properties:

* `messageId`: the ID of the packet
* `subscriptions`: an array of objects
representing the subscribed topics, containing the following keys
  * `topic`: the topic subscribed to
  * `qos`: the qos level of the subscription


#### Event: 'suback'
`function(packet) {}`

Emitted when a MQTT suback packet is received.

`packet` is an object that may contain the properties:

* `messageId`: the ID of the packet
* `granted`: a vector of granted QoS levels

#### Event: 'unsubscribe'
`function(packet) {}`

Emitted when a MQTT unsubscribe packet is received.

`packet` is an object that may contain the properties:

* `messageId`: the ID of the packet
* `unsubscriptions`: a list of topics the client is
unsubscribing from, of the form `[topic1, topic2, ...]`

#### Events: \<'pingreq', 'pingresp', 'disconnect'\>
`function(packet){}`

Emitted when a MQTT `[pingreq, pingresp, disconnect]` packet is received.

`packet` only includes static header information and can be ignored.

-------------------------------------

<a name="generateStream">

### mqtt.generateStream()

Returns a `Transform` stream that calls [`generate()`](https://github.com/mqttjs/mqtt-packet#generate).
The stream is configured into object mode.

<a name="parseStream">

### mqtt.parseStream(opts)

Returns a `Transform` stream that embeds a [`Parser`](https://github.com/mqttjs/mqtt-packet#mqttparser) and calls [`Parser.parse()`](https://github.com/mqttjs/mqtt-packet#parserparsebuffer) for each new `Buffer`. The stream is configured into object mode. It accepts the same options of [`parser(opts)`](#parser).

<a name="contributing"></a>
Contributing
------------

mqtt-connection is an **OPEN Open Source Project**. This means that:

> Individuals making significant and valuable contributions are given commit-access to the project to contribute as they see fit. This project is more like an open wiki than a standard guarded open source project.

See the [CONTRIBUTING.md](https://github.com/mqttjs/mqtt-connection/blob/master/CONTRIBUTING.md) file for more details.

### Contributors

mqtt-connection is only possible due to the excellent work of the following contributors:

<table><tbody>
<tr><th align="left">Matteo Collina</th><td><a href="https://github.com/mcollina">GitHub/mcollina</a></td><td><a href="http://twitter.com/matteocollina">Twitter/@matteocollina</a></td></tr>
<tr><th align="left">Adam Rudd</th><td><a href="https://github.com/adamvr">GitHub/adamvr</a></td><td><a href="http://twitter.com/adam_vr">Twitter/@adam_vr</a></td></tr>
</tbody></table>

License
-------

MIT
