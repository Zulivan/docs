!!! tip "Colyseus Cloud"
    If you find configuring this overwhelming, consider using [Colyseus Cloud](https://cloud.colyseus.io/) to easily deploy and host your Colyseus servers.

### How does Colyseus achieve scalability?

- Increasing the number of processes will increase the amount of rooms that can be created.
- Rooms are equally distributed across the available processes.
- Each Room belongs to a single Colyseus process.
- Each process **has a maximum amount of players** it could handle. _(The exact amount depends on many factors. [See our FAQ](/faq/#how-many-ccu-a-colyseus-server-can-handle).)_
- Client connections are directly associated with the process that created the room.

### There are two strategies available for scaling:

- **Recommended:** Making every Node.js process accessible to the internet via a public URL. This way clients can connect directly to the process that created the room.
- Previous recommendation (_not a recommendation since 0.15_): Use `@colyseus/proxy` behind every Node.js process that redirects client traffic to the correct process.

!!! Note "Redis is required"
    [Redis](https://redis.io/topics/quickstart) is required for both strategies. Make sure to have it running on your server.

## Step 1: Use set-up `Presence` and a shared `Driver`

=== "TypeScript"

    ``` typescript
    import { Server } from "colyseus";
    import { RedisPresence } from "@colyseus/redis-presence";
    import { RedisDriver } from "@colyseus/redis-driver";

    const gameServer = new Server({
      // ...
      presence: new RedisPresence(),
      driver: new RedisDriver(),
    });
    ```

=== "JavaScript"

    ``` typescript
    const colyseus = require("colyseus");
    const { RedisPresence } = require("@colyseus/redis-presence");
    const { RedisDriver } = require("@colyseus/redis-driver");

    const gameServer = new colyseus.Server({
      // ...
      presence: new colyseus.RedisPresence(),
      driver: new colyseus.RedisDriver(),
    });
    ```

The `presence` is used to call room "seat reservation" functions from one process to another. If you are using servers with different IPs, make sure each server uses its own distinct database id for the presence. See [Presence API](/server/presence/#api).

The `driver` is responsible for keeping a shared cache of available rooms, and room querying by any of the active Colyseus processes.

## Step 2: Making your Colyseus processes accessible to the internet

### Recommendation: Using a Load Balancer

Since version 0.15, you can configure each Colyseus process to use its very own public address, so clients can connect directly to it.

``` typescript
const server = new Server({
    // ...
    presence: new RedisPresence(),
    driver: new RedisDriver(),

    // use a unique public address for each process
    publicAddress: "server-1.yourdomain.com"
});
```

You can have a regular load balancer sitting behind all the Colyseus processes. The load balancer must be the initial entrypoint for all connections.

### Previous recommendation: Using a Dynamic Proxy

!!! Warning "Proxy is not a recommendation on version 0.15"
    Please check the load balancer recommendation above.

Install the [@colyseus/proxy](https://github.com/colyseus/proxy).

``` bash
npm install --save @colyseus/proxy
```

The Dynamic Proxy automatically listens whenever a Colyseus process goes up and down. It is responsible for routing the client requests to the correct Colyseus process.

All client requests must use the proxy as endpoint. Using this solution, clients do not communicate **directly** with the server, but through the proxy.

The proxy should be bound to port `80` / `443` on a production environment.

**Environment variables**

Configure the following environment variables to meet your needs:

- `PORT` is the port the proxy will be running on.
- `REDIS_URL` is the path to the same Redis instance you're using on Colyseus' processes.

**Running the proxy**

```
npx colyseus-proxy

> {"name":"redbird","hostname":"Endels-MacBook-Air.local","pid":33390,"level":30,"msg":"Started a Redbird reverse proxy server on port 80","time":"2019-08-20T15:26:19.605Z","v":0}
```

## Step 3: Spawning multiple Colyseus processes

To run multiple Colyseus instances in the same server, you need each one of them to listen on a different port number. It's recommended to use ports `3001`, `3002`, `3003`, and so on.

- If you're [using a load balancer (recommendation)](#recommendation-using-a-load-balancer), each Colyseus process **MUST** have its own public address.
- If you're [using `@colyseus/proxy` (not recommended)](#previous-recommendation-using-a-dynamic-proxy), the Colyseus processes should **NOT** be exposed publicly. But only internally for the proxy.

The [PM2 process manager](http://pm2.keymetrics.io/) is recommended for managing multiple Node.js app instances, although not mandatory.

PM2 provides a `NODE_APP_INSTANCE` environment variable, containing a different number for each process. Use it to define your port number.

``` typescript
import { Server } from "colyseus";

// binds each instance of the server on a different port.
const PORT = Number(process.env.PORT) + Number(process.env.NODE_APP_INSTANCE);

const gameServer = new Server({ /* ... */ })

gameServer.listen(PORT);
console.log("Listening on", PORT);
```

``` bash
npm install -g pm2
```

Use the following `ecosystem.config.js` configuration:

``` javascript
// ecosystem.config.js
const os = require('os');
module.exports = {
    apps: [{
        port        : 3000,
        name        : "colyseus",
        script      : "lib/index.js", // your entrypoint file
        watch       : true,           // optional
        instances   : os.cpus().length,
        exec_mode   : 'fork',         // IMPORTANT: do not use cluster mode.
        env: {
            DEBUG: "colyseus:errors",
            NODE_ENV: "production",
        }
    }]
}
```

Now you're ready to start multiple Colyseus proceses.

``` bash
pm2 start
```

!!! Note "PM2 and TypeScript"
    It's recommended compile your .ts files before running `pm2 start`, via `npx tsc`. Alternatively, you can install the TypeScript interpreter for PM2 (`pm2 install typescript`) and set the `exec_interpreter: "ts-node"` ([read more](http://pm2.keymetrics.io/docs/tutorials/using-transpilers-with-pm2)).
