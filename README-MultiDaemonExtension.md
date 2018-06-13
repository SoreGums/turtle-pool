# Turtle-Pool Multi Daemon Mode
## What?
This is an extension to the turtle-pool software to provide stable and reliable access to the TRTL Network by exposing multiple daemons to the pool software. There are smarts included to select an appropriate daemon, operators will still need to monitor their pools. This extension will simply provide another tool to assist with the task and ensure 24/7 operations.

Extension means it can be turned off and on. This code can live inside the current turtle-pool software stack and have zero impact on the standard 1 pool, 1 daemon mode of operation. If turned on, then this extension will do its thing and monitor multiple daemons for an appropriate one to use.

## Why?
Right now *(2018-05-22)* the TurtleCoind C++ daemon has stability issues. The issues are related to transaction processing at the root. This means that when a certain kind of transaction is broadcast on the network nodes have trouble processing it. Then the transaction makes its way into a block, the block gets broadcast and then nodes again have issues processing.  

Not all nodes have issues all the time, which is what makes this issue problematic. Not all daemons crash. There are also other stability issues, however, the actual processing of the transaction is the beginning of the problems. Which then causes problems in the storage system with trying to verify everything and storage issues that crop up there. Possible reasons for why this is an issue that affects TRTL Network and not others are due to the fast block time. Fast blocks = processing more often, then the system gets behind, then it tries to store things wherever temporarily, then things become unstable due to the code having never been designed and tested for such a situation.
**Which brings us to the point.**  
When a mining pool operator runs the pool software the pool needs to communicate with the TRTL Network, it does this via TurtleCoind. If TurtleCoind stops responding:

 - the miners can't mine
 - the pools hashrate is no longer contributing to the network, depending on the hashrate of the pool, this can be a major problem for the stability and reliability of the network.
 - everything the pool exists for stops being a thing.

Ensuring the TurtleCoind daemon runs 24/7 with zero issues isn't the goal. The actual goal is to **maintain a connection to the TRTL Network** that the pool software can use to do its thing is the actual goal.  

So how do pool operators keep the connection to the network? By ensuring the daemon is running. There are many ways to ensure the daemon is running some of them are:

- Run the whole operation in kubernettes
	- This is the best choice, however, kubernettes is NOT simple, the reason it is the best choice is that it works when configured and deployed completely as k8s is designed to work. Zero to 100 is probably a month or more of work/study/practise, so this is pretty unapproachable for a lot of people.
- Have a resource available to watch the TurtleCoind daemon and restart it when there are issues.
	- This is how it is mostly done, a resource can be anything, look up a resource for the verb.
	- No human wants to babysit a console and press keys as required
	- Have a computer do the job, this is what they are good for.
		- iburnmycd has built out a tool to keep a daemon running and it works great check it out: [turtlecoind-ha](https://github.com/turtlecoin/turtlecoind-ha). What it doesn't do is verify if the daemon running is actually in sync with the network or not. iburnmycd had plans to figure this out, however, that was put on hold recently to pursue the higher goal of making a stable daemon thus negating the need for such a tool ;)
		- write scripts to do things, this is the bread and butter of sysadmins everywhere. However this doesn't help all the time due to some quirks of the daemon and other areas of its unstableness. The gap is why this extension exists.

The above solutions all involve doing things outside of the pool software and having expertise in those external areas. This extension to the current pool software will bring some of that skill to the software. The operator will still need to run daemons and put things in place to ensure they restart etc

### Summary on Why?
Mining Pools need a way to maintain their connection to the TRTL Network, this extension to turtle-pool exists to solve that by allowing the pool operator to configure multiple daemons. By having access to multiple daemons the pool can be operational 24/7 with minimal unscheduled downtime. Provided the operator can run multiple daemons and keep them restarting.

This is all a temporary fix until a stable daemon comes along, so why is SoreGums doing this then? For the experience and to explore another aspect of the blockchain technology. It will also serve as guidance/talking point for any daemons that are created in the future. Specifically in terms of monitoring network health and what kinds of actions are appropriate to automate. There a ton of gotchas in attempting this kind of a solution and they are explored in the How section below.

## How?

### Enable/Config it
Config for this is simple, current `daemon` setting remains. This will be identified as the primary daemon. Then a new block is added called `mutliDaemon` and looks like this:
```json
"multiDaemon": {
  "enabled": true,
  "daemons": [
    {
      "host": "127.0.0.1",
      "port": 11500
    },
    {
      "host": "127.0.0.1",
      "port": 11600
    },
    {
      "host": "127.0.0.1",
      "port": 11700
    }
  ]
}
```

### Logic/Daemon Selection
For this to work reliably, minimum of 3 active *working* daemons are required, lack of 3 will turn this feature off and go back to the regular `1x pool, 1x daemon` operation mode. That 1x daemon will be the primary one as defined in the config.

There could be 7 daemons configured in the pool, this means 4x of them could be offline, either stuck or being restarted, resyncing etc. As long as 3x of the 7x are responding properly `healthy` and not marked as `unhealthy` this extension will be active.

#### 3x Daemons?

The 3x includes the primary one. So if two daemons are defined in multiDaemon block, then there are three daemons. That's not ideal cause as soon as any of the daemons die this extension turns off and things go back to  `1x pool, 1x daemon` operation mode.

### Implementation Specifics
Each daemon can have any one of these statuses:

- **healthy**: daemon is ok to be used by the pool
- **unhealthy**: daemon not ok for some reason and can't be used by the pool, it is contactable and responding to API requests, however there is an issue that has been detected
- **unresponsive**: daemon no longer responding to RPC requests, as such can't be used by the pool

#### Determining Healthy Daemons
Finally the nuts and bolts.

The steps are simple and logical. All the daemons added to the pool are assumed to be functioning daemons, connected to the regular 8 outgoing peers and if incoming is enabled, n incoming peers, doing network node blockchain things *(replicating transactions & blocks, validating the chain)*. As such if they report **in_sync: true** when doing `getInfo` the first time, then they are in sync with the network and the node is healthy. First check out of the way, baseline established.

So all nodes are fine, now this extensions logic takes over and determines which daemon the pool will use.

The primary daemon is also used initially to set the baseline, its block height and hash of that block will be assumed to be correct. So that initial `getInfo` call will also get the current block's hash, plus the previous 3x block's hashes. All daemons should be on the same chain and hopefully at the same current block. the reason for getting the last 4 blocks is in case network conditions are causing a delay. if daemons are on the same chain and behind they will be marked as `unhealthy` with a reason of `on same chain, lagging behind other nodes`.

Right have got consensus at this point, either all nodes are the same as primary or lagging or in front. From here the call can be made to select the current daemon. Now to signal to the modules involved which daemon to use, specifically the IP and Port.

#### Check 1 - blockchain
On some interval, probably every 5 seconds, unless turtlecoind-ha is used then can sub to events, each daemon will be pinged. Block chains compared *(height, hash, previous 3 hashes)*, the majority wins. Any daemon not in agreement gets marked `unhealthy` with a reason and daemon that doesn't respond gets marked `unresponsive`.

#### Check 2 - peers
While the first check is pretty much it. there is another one. If the configured daemons only talk with each other they are not really part of the wider TRTL Network, so would want to mark those daemons as unhealthy as well.

**Needs more words about how the checks are done and which daemon is picked**

#### Pathway 1 - Dynamic Config feature (This README written before code, got 2 options. the 1st)
Rewrite turtle-pool to work with a dynamic config, this also creates a path for zero downtime config updates.
That's it, if the config is dynamic, then just update the config.

#### Pathway 2 - When this module is active, read dynamic config for daemon info.
Requires some javascript-jitsu

## Objections To This Whole Idea
> This is stupid, doing this you're risking the health of the whole network, stahp.

OK, got any specifics? Below are some objections that came up when talking to others about this idea. Most of them are about automatically selecting the daemon, however each one I responded thusly:

> well this daemon that you're saying is worse, that could be picked, could have been the actual daemon in the first place and the original issue still exists = pool doesn't give useable work to miners, pool hashrate not going towards network's main chain. This project doesn't alter how TurtleCoind daemon works, all of its quirtks still exist, the point is to ensure pool is connected to the network, specifically the main chain, hence all the checks described above in How?

#### Selecting the daemon based on height is dumb.
Agreed, that's why it is not the decieding factor, as above, it is the first way to figure out where things are at. Then block hashes of the viable candidates are compared. Read above for details, not writing it again here :/

####  Your going to pick a forked daemon and cause more problems.
Well that is a risk, see the intro response. Hopefully this doesn't happen, if it does it CAN'T be mitigated in the first place. The pool operator still needs to *operate* their pool even if they run this extension.

#### Any pool running this is going to amplify Timewarp attacks
You wot m8? r u 'avin a giggle, M8?
That's not how timewarps work. Even if pulsing is actually being referred to here, the OP miner will move to the next pool and bring that one down, thus making the whole endeavor easier as pools get knowcked off the network thus reducing the network hashrate.

#### Anything else?

> Written with [StackEdit](https://stackedit.io/).
