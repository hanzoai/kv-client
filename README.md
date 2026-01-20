# @hanzo/kv-client

A robust, performance-focused and full-featured client for [Hanzo KV](https://hanzo.ai), [Valkey](https://valkey.io), and [Redis](https://redis.io) in [Node.js](https://nodejs.org).

This package is part of the [Hanzo AI](https://hanzo.ai) ecosystem and is a fork of [iovalkey](https://github.com/valkey-io/iovalkey).

Supports Hanzo KV, Valkey >= 2.6.12, and Redis. Completely compatible with Valkey 7.x and Redis 7.x.

## Installation

```shell
npm install @hanzo/kv-client
```

In a TypeScript project, you may want to add TypeScript declarations for Node.js:

```shell
npm install --save-dev @types/node
```

## Features

@hanzo/kv-client is a robust, full-featured KV client that powers the Hanzo AI infrastructure.

0. Full-featured. Supports [Cluster](http://valkey.io/topics/cluster-tutorial), [Sentinel](https://valkey.io/docs/reference/sentinel-clients), [Streams](https://valkey.io/topics/streams-intro), [Pipelining](http://valkey.io/topics/pipelining), [Lua scripting](http://valkey.io/commands/eval), [Valkey Functions](https://valkey.io/topics/functions-intro), [Pub/Sub](http://valkey.io/topics/pubsub) (with binary message support).
1. High performance 🚀.
2. Delightful API 😄. Works with Node callbacks and Native promises.
3. Transformation of command arguments and replies.
4. Transparent key prefixing.
5. Abstraction for Lua scripting, allowing you to define custom commands.
6. Supports binary data.
7. Supports TLS 🔒.
8. Supports offline queue and ready checking.
9. Supports ES6 types, such as `Map` and `Set`.
10. Supports GEO commands 📍.
11. Supports Valkey/Redis ACL.
12. Sophisticated error handling strategy.
13. Supports NAT mapping.
14. Supports autopipelining.

**100% written in TypeScript and official declarations are provided.**

## Quick Start

```javascript
// Import @hanzo/kv-client
// TypeScript: import { HanzoKV } from "@hanzo/kv-client"
// or: import HanzoKV from "@hanzo/kv-client"
const HanzoKV = require("@hanzo/kv-client");

// Create a client instance
// By default, it connects to localhost:6379
const kv = new HanzoKV();

kv.set("mykey", "value"); // Returns a promise which resolves to "OK"

// Callback style
kv.get("mykey", (err, result) => {
  if (err) {
    console.error(err);
  } else {
    console.log(result); // Prints "value"
  }
});

// Promise style
kv.get("mykey").then((result) => {
  console.log(result); // Prints "value"
});

kv.zadd("sortedSet", 1, "one", 2, "dos", 4, "quatro", 3, "three");
kv.zrange("sortedSet", 0, 2, "WITHSCORES").then((elements) => {
  console.log(elements);
});

// All arguments are passed directly to the server
kv.set("mykey", "hello", "EX", 10);
```

## Connect to Hanzo KV / Valkey / Redis

```javascript
new HanzoKV(); // Connect to 127.0.0.1:6379
new HanzoKV(6380); // 127.0.0.1:6380
new HanzoKV(6379, "192.168.1.1"); // 192.168.1.1:6379
new HanzoKV("/tmp/hanzo-kv.sock");
new HanzoKV({
  port: 6379,
  host: "127.0.0.1",
  username: "default", // needs server >= 6
  password: "my-top-secret",
  db: 0,
});
```

Using connection URLs:

```javascript
// Connect to 127.0.0.1:6380, db 4, using password "authpassword":
new HanzoKV("redis://:authpassword@127.0.0.1:6380/4");

// Username can also be passed via URI
new HanzoKV("redis://username:authpassword@127.0.0.1:6380/4");
```

## Pub/Sub

```javascript
// publisher.js
const HanzoKV = require("@hanzo/kv-client");
const kv = new HanzoKV();

setInterval(() => {
  const message = { foo: Math.random() };
  const channel = `my-channel-${1 + Math.round(Math.random())}`;
  kv.publish(channel, JSON.stringify(message));
  console.log("Published %s to %s", message, channel);
}, 1000);
```

```javascript
// subscriber.js
const HanzoKV = require("@hanzo/kv-client");
const kv = new HanzoKV();

kv.subscribe("my-channel-1", "my-channel-2", (err, count) => {
  if (err) {
    console.error("Failed to subscribe: %s", err.message);
  } else {
    console.log(`Subscribed to ${count} channels.`);
  }
});

kv.on("message", (channel, message) => {
  console.log(`Received ${message} from ${channel}`);
});
```

## Streams

```javascript
const HanzoKV = require("@hanzo/kv-client");
const kv = new HanzoKV();

const processMessage = (message) => {
  console.log("ID: %s. Data: %O", message[0], message[1]);
};

async function listenForMessage(lastID = "$") {
  const results = await kv.xread("block", 0, "STREAMS", "mystream", lastID);
  const [key, messages] = results[0];
  messages.forEach(processMessage);
  await listenForMessage(messages[messages.length - 1][0]);
}

listenForMessage();
```

## Pipelining

```javascript
const pipeline = kv.pipeline();
pipeline.set("foo", "bar");
pipeline.del("cc");
pipeline.exec((err, results) => {
  // results is an array of responses
});

// Chain commands
kv.pipeline()
  .set("foo", "bar")
  .del("cc")
  .exec((err, results) => {});

// Returns a Promise
const promise = kv.pipeline().set("foo", "bar").get("foo").exec();
promise.then((result) => {
  // result === [[null, 'OK'], [null, 'bar']]
});
```

## Transaction

```javascript
kv.multi()
  .set("foo", "bar")
  .get("foo")
  .exec((err, results) => {
    // results === [[null, 'OK'], [null, 'bar']]
  });
```

## Lua Scripting

```javascript
const kv = new HanzoKV();

kv.defineCommand("myecho", {
  numberOfKeys: 2,
  lua: "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}",
});

kv.myecho("k1", "k2", "a1", "a2", (err, result) => {
  // result === ['k1', 'k2', 'a1', 'a2']
});

// Works with pipeline
kv.pipeline().set("foo", "bar").myecho("k1", "k2", "a1", "a2").exec();
```

## Cluster

```javascript
const HanzoKV = require("@hanzo/kv-client");

const cluster = new HanzoKV.Cluster([
  { port: 6380, host: "127.0.0.1" },
  { port: 6381, host: "127.0.0.1" },
]);

cluster.set("foo", "bar");
cluster.get("foo", (err, res) => {
  // res === 'bar'
});
```

### Read-Write Splitting

```javascript
const cluster = new HanzoKV.Cluster(
  [/* nodes */],
  { scaleReads: "slave" }
);
cluster.set("foo", "bar"); // Sent to master
cluster.get("foo", (err, res) => {
  // Sent to slave
});
```

## Sentinel

```javascript
const kv = new HanzoKV({
  sentinels: [
    { host: "localhost", port: 26379 },
    { host: "localhost", port: 26380 },
  ],
  name: "mymaster",
});

kv.set("foo", "bar");
```

## Auto-reconnect

```javascript
const kv = new HanzoKV({
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
});
```

## TLS Options

```javascript
const kv = new HanzoKV({
  host: "localhost",
  tls: {
    ca: fs.readFileSync("cert.pem"),
  },
});
```

Or via URL:

```javascript
const kv = new HanzoKV("rediss://kv.my-service.com");
```

## Autopipelining

Enable automatic pipelining for improved throughput:

```javascript
const kv = new HanzoKV({ enableAutoPipelining: true });
```

In auto pipelining mode, all commands issued during an event loop are enqueued in a pipeline automatically. This can improve throughput by 35-50%.

## Connection Events

| Event        | Description                                                    |
| :----------- | :------------------------------------------------------------- |
| connect      | Connection established to the server                           |
| ready        | Server ready to receive commands                               |
| error        | Error while connecting                                         |
| close        | Connection closed                                              |
| reconnecting | Reconnection will be made                                      |
| end          | No more reconnections will be made                             |
| wait         | lazyConnect is set, waiting for first command                  |

## Error Handling

```javascript
const HanzoKV = require("@hanzo/kv-client");
const kv = new HanzoKV();

kv.set("foo", (err) => {
  err instanceof HanzoKV.ReplyError;
});
```

Enable friendly error stack for debugging:

```javascript
const kv = new HanzoKV({ showFriendlyErrorStack: true });
```

## Running Tests

Start a Hanzo KV/Valkey/Redis server on 127.0.0.1:6379, then:

```shell
npm test
```

## Debug

```shell
$ DEBUG=hanzo-kv:* node app.js
```

## Hanzo Ecosystem

@hanzo/kv-client is part of the Hanzo AI infrastructure:

- **[Hanzo KV](https://github.com/hanzoai/kv)** - High-performance key-value server (Valkey fork)
- **[@hanzo/mq](https://github.com/hanzoai/mq)** - Message queue built on Hanzo KV
- **[Hanzo Auto](https://github.com/hanzoai/auto)** - Workflow automation platform
- **[Hanzo Flow](https://github.com/hanzoai/flow)** - Visual AI/LLM workflow builder

## License

MIT

## Credits

This package is based on [iovalkey](https://github.com/valkey-io/iovalkey), which was forked from [ioredis](https://github.com/redis/ioredis).
