<div align="center">
  <p>
  <a href="https://github.com/OceanicJS/Oceanic"><img src="https://img.shields.io/badge/Discord%20Library-Oceanic-blue?style=flat-square" alt="Discord Library" /></a>
    <a href="https://raw.githubusercontent.com/esmBot/oceanic-fleet/master/LICENSE"><img alt="License" src="https://img.shields.io/npm/l/eris-fleet?style=flat-square"></a>
    <a href="https://github.com/esmBot/oceanic-fleet/actions/workflows/ci.yml"><img src="https://img.shields.io/github/workflow/status/esmBot/oceanic-fleet/Node.js%20CI/master?style=flat-square&logo=github" alt="Node.js CI" /></a>
  </p>
</div>

### [Documentation](https://esmbot.github.io/oceanic-fleet/) | [Oceanic](https://github.com/OceanicJS/Oceanic)

# About oceanic-fleet

A fork of danclay's [eris-fleet](https://github.com/danclay/eris-fleet) designed to work with the newer [Oceanic](https://github.com/OceanicJS/Oceanic) library, which was created in response to the slow and difficult maintenance of Eris.

eris-fleet is a spin-off of [eris-sharder](https://github.com/discordware/eris-sharder) and [megane](https://github.com/brussell98/megane) with services and configurable logging.

For detailed documentation check the [docs](https://esmbot.github.io/oceanic-fleet/).

oceanic-fleet currently supports Oceanic v1.0.0.

## Highlighted Features:

- Clustering across cores
- Sharding
- Recalculate shards with minimal downtime
- Update a bot with minimal downtime using soft restarts
- Customizable logging
- Fetch data from across clusters easily
- Services (non-oceanic workers)
- IPC to communicate between clusters, other clusters, and services
- Detailed stats collection
- Soft cluster and service restarts where the old worker is killed after the new one is ready
- Graceful shutdowns
- Central request handler
- Central data store
- Can use a modified version of Oceanic
- Concurrency support

A very basic diagram:

![Basic diagram](https://cdn.discordapp.com/attachments/866590047436144641/965161462479859742/Untitled_Diagram.drawio_1.png)

## Help

If you still have questions, you can join the esmBot Support server: https://discord.gg/esmbot

[![Support server on Discord](https://discord.com/api/guilds/592399417676529688/widget.png?style=banner2)](https://discord.gg/esmbot)

# Installation
Run `npm install eris-fleet` or with yarn: `yarn add eris-fleet`.

To use a less refined, but more up-to-date branch, use `npm install esmBot/oceanic-fleet#dev` or `yarn add esmBot/oceanic-fleet#dev`. [Documentation for the dev branch.](https://github.com/esmBot/oceanic-fleet/tree/dev)

# Basics

Some working examples are in [test/](https://github.com/esmBot/oceanic-fleet/tree/master/test).

## Naming Conventions
| Term | Description |
|-----------|----------------------------------------------------------------------------|
| "fleet" | All the components below |
| "admiral" | A single sharding manager |
| "worker" | A worker for node clustering |
| "cluster" | A worker containing Oceanic shards |
| "service" | A worker that does not contain Oceanic shards, but can interact with clusters |

## Get Started
To get started, you will need at least 2 files:
1. Your file which will create the fleet. This will be called "index.js" for now.
2. Your file containing your bot code. This will be called "bot.js" for now. This file will extend `BaseClusterWorker`

In the example below, the variable `options` is passed to the admiral. [Read the docs](https://esmbot.github.io/oceanic-fleet/interfaces/Options.html) for what options you can pass.

Here is an example of `index.js`:
```js
const { isPrimary } = require('cluster');
const { Fleet } = require('oceanic-fleet');
const path = require('path');
const { inspect } = require('util');

require('dotenv').config();

const options = {
    path: path.join(__dirname, "./bot.js"),
    token: process.env.token
}

const Admiral = new Fleet(options);

if (isPrimary) {
    // Code to only run for your master process
    Admiral.on('log', m => console.log(m));
    Admiral.on('debug', m => console.debug(m));
    Admiral.on('warn', m => console.warn(m));
    Admiral.on('error', m => console.error(inspect(m)));

    // Logs stats when they arrive
    Admiral.on('stats', m => console.log(m));
}
```
This creates a new Admiral that will manage `bot.js` running in other processes. [More details](https://esmbot.github.io/oceanic-fleet/classes/BaseClusterWorker.html)

The following is an example of `bot.js`. [Read the IPC docs](https://esmbot.github.io/oceanic-fleet/classes/IPC.html) for what you can access and do with clusters.
```js
const { BaseClusterWorker } = require('oceanic-fleet');

module.exports = class BotWorker extends BaseClusterWorker {
    constructor(setup) {
        // Do not delete this super.
        super(setup);

        this.bot.on('messageCreate', this.handleMessage.bind(this));

        // Demonstration of the properties the cluster has (Keep reading for info on IPC):
        this.ipc.log(this.workerID); // ID of the worker
        this.ipc.log(this.clusterID); // The ID of the cluster
    }

    async handleMessage(msg) {
        if (msg.content === "!ping" && !msg.author.bot) {
            this.bot.rest.channels.createMessage(msg.channelID, {content: "Pong!"});
        }
    }

	handleCommand(dataSentInCommand) {
		// Optional function to return data from this cluster when requested
		return "hello!"
	}

    shutdown(done) {
        // Optional function to gracefully shutdown things if you need to.
        done(); // Use this function when you are done gracefully shutting down.
    }
}
```
**Make sure your bot file extends BaseClusterWorker!**
The bot above will respond with "Pong!" when it receives the command "!ping".

## Services

You can create services for your bot. Services are workers which do not interact directly with Oceanic. Services are useful for processing tasks, a central location to get the latest version of languages for your bot, custom statistics, and more! [Read the IPC docs](https://esmbot.github.io/oceanic-fleet/classes/IPC.html) for what you can access and do with services. **Note that services always start before the clusters. Clusters will only start after all the services have started.** [More details](https://esmbot.github.io/oceanic-fleet/classes/BaseServiceWorker.html)

To add a service, add the following to the options you pass to the fleet:

```js
const options = {
    // Your other options...
    services: [{name: "myService", path: path.join(__dirname, "./service.js")}]
}
```
Add a new array element for each service you want to register. Make sure each service has a unique name or else the fleet will crash.

Here is an example of `service.js`:

```js
const { BaseServiceWorker } = require('oceanic-fleet');

module.exports = class ServiceWorker extends BaseServiceWorker {
    constructor(setup) {
        // Do not delete this super.
        super(setup);

        // Run this function when your service is ready for use. This MUST be run for the worker spawning to continue.
        this.serviceReady();

        // Demonstration of the properties the service has (Keep reading for info on IPC):
    	this.ipc.log(this.workerID); // ID of the worker
    	this.ipc.log(this.serviceName); // The name of the service

    }
    // This is the function which will handle commands
    async handleCommand(dataSentInCommand) {
        // Return a response if you want to respond
        return dataSentInCommand.smileyFace;
    }

    shutdown(done) {
        // Optional function to gracefully shutdown things if you need to.
        done(); // Use this function when you are done gracefully shutting down.
    }
}
```

**Make sure your service file extends BaseServiceWorker!**
This service will simply return a value within an object sent to it within the command message called "smileyFace". Services can be used for much more than this though. To send a command to this service, you could use this:

```js
const reply = await this.ipc.command("myService", {smileyFace: ":)"}, true);
this.bot.rest.channels.createMessage(msg.channelID, {content: reply});
```

This command is being sent using the IPC. In this command, the first argument is the name of the service to send the command to, the second argument is the message to send it (in this case a simple object), and the third argument is whether you want a response (this will default to false unless you specify "true"). If you want a response, you must `await` the command or use `.then()`.

### Handling service errors

If you encounter an error while starting your service, run `this.serviceStartingError('error here')` instead of `this.serviceReady()`. Using this will report an error and restart the worker. **Note that services always start before the clusters, so if your service keeps having starting errors your bot will be stuck in a loop.** This issue may be fixed in the future from some sort of maxRestarts option, but this is currently not a functionality.

If you encounter an error when processing a command within your service, you can do the following to reject the promise:
```js
// handleCommand function within the ServiceWorker class
async handleCommand(dataSentInCommand) {
    // Rejects the promise
    return {err: "Uh oh.. an error!"};
}
```
When sending the command, you can do the following to deal with the error:
```js
this.ipc.command("myService", {smileyFace: ":)"}, true).then((reply) => {
    // A successful response
    this.bot.rest.channels.createMessage(msg.channelID, {content: reply});
}).catch((e) => {
    // Do whatever you want with the error
    console.error(e);
});
```

# In-depth

Below is more in-depth documentation.

## Admiral 

### Admiral options

Visit [the docs](https://esmbot.github.io/oceanic-fleet/interfaces/Options.html) for a complete list of options.

### Admiral events

Visit [the docs](https://esmbot.github.io/oceanic-fleet/classes/Fleet.html) for a complete list of events.

### Central Request Handler

The central request handler forwards Oceanic requests to the master process where the request is sent to a single Oceanic request handler instance. This helps to prevent 429 errors from occurring when you have x number of clusters keeping track of ratelimiting separately. When a response is received, it is sent back to the cluster's Oceanic client.

### Large Bots

If you are using a "very large bot," Discord's special gateway settings apply. Ensure your shard count is a multiple of the number set by Discord or set `options.shards` and `options.guildsPerShard` to `"auto"`. You may also be able to use concurrency (see below).

### Concurrency

oceanic-fleet supports concurrency by starting clusters at the same time based on your bot's `max_concurrency` value. The clusters are started together in groups. The `max_concurrency` value can be overridden with [options.maxConcurrencyOverride](https://esmbot.github.io/oceanic-fleet/interfaces/Options.html#maxConcurrencyOverride)

### Formats

Visit [the docs](https://esmbot.github.io/oceanic-fleet/modules.html) to view the Typescript interfaces.

### Choose what to log

You can choose what to log by using the `whatToLog` property in the options object. You can choose either a whitelist or a blacklist of what to log. You can select what to log by using an array. To possible array elements are shown [on the docs](https://esmbot.github.io/oceanic-fleet/modules.html#LoggingOptions). Here is an example of choosing what to log:
```js
const options = {
    // Your other options
    whatToLog: {
        // This will only log when the admiral starts, when clusters are ready, and when services are ready.
        whitelist: ['admiral_start', 'cluster_ready', 'service_ready']
    }
};
```
Change `whitelist` to `blacklist` if you want to use a blacklist. Change the array as you wish. **Errors and warnings will always be sent.**

## IPC

Clusters and services can use IPC to interact with other clusters, the Admiral, and services. Visit [the IPC docs](https://esmbot.github.io/oceanic-fleet/classes/IPC.html) to view available methods.

## Stats

Stats are given in [this](https://esmbot.github.io/oceanic-fleet/interfaces/Stats.html) format.

## Using a specific version of Oceanic or a modified version of Oceanic

You can use an extended Oceanic client by passing it to the Options. (see the [options.customClient](https://esmbot.github.io/oceanic-fleet/interfaces/Options.html#customClient) section).

## Using ES Modules

Instead of using the file path, you can use ES Modules by passing your BotWorker class to `options.BotWorker` and your ServiceWorker class to `ServiceWorker` in the `options.services` array. See [test/](https://github.com/esmbot/oceanic-fleet/tree/master/test) for examples.