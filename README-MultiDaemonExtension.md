# Turtle-Pool Multi Daemon Mode
## What?
This is an extension to the turtle-pool software to provide stable and reliable access to the TurtleCoin network by exposing multiple daemons to the pool software. There are smarts included to select an appropriate daemon, operators will still need to monitor their pools. This extension will simply provide another tool to assist with the task and ensure 24/7 operations.

Extension means it can be turned off and on. This code can live inside the current turtle-pool software stack and have zero impact on the standard 1 pool, 1 daemon mode of operation. If turned on, then this extension will do its thing and monitor multiple daemons for an appropriate one to use.

## Why?
Right now *(2018-05-22)* the TurtleCoind C++ daemon has stability issues. The issues are related to transaction processing at the root. This means that when a certain kind of transaction is broadcast on the network nodes have trouble processing it. Then the transaction makes its way into a block, the block gets broadcast and then nodes again have issues processing.  

Not all nodes have issues all the time, which is what makes this issue problematic. Not all daemons crash. There are also other stability issues, however, the actual processing of the transaction is the beginning of the problems. Which then causes problems in the storage system with trying to verify everything and store issues that crop up there. Possible reasons for why this is an issue that affects TurtleCoin and not others are due to the fast block time. Fast blocks = processing more often, then the system gets behind, then it tries to store things wherever temporarily, then things become unstable due to the code having never been designed and tested for such a situation.  

**Which brings us to the point.**  
When a mining pool operator runs the pool software the pool needs to communicate with the TurtleCoin network, it does this via TurtleCoind. If TurtleCoind stops responding:

 - the miners can't mine
 - the pools hashrate is no longer contributing to the network, depending on the hashrate of the pool, this can be a major problem for the stability and reliability of the network.
 - everything the pool exists for stops being a thing.

Ensuring the TurtleCoind daemon runs 24/7 with zero issues isn't the goal. The actual goal is to **maintain a connection to the TurtleCoin blockchain network** that the pool software can use to do its thing is the actual goal.  

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
Mining Pools need a way to maintain their connection to the TurtleCoin network, this extension to turtle-pool exists to solve that by allowing the pool operator to configure multiple daemons. By having access to multiple daemons the pool can be operational 24/7 with minimal unscheduled downtime. Provided the operator can run multiple daemons and keep them restarting.

This is all a temporary fix until a stable daemon comes along, so why is SoreGums doing this then? For the experience and to explore another aspect of the blockchain technology. It will also serve as guidance/talking point for any daemons that are created in the future. Specifically in terms of monitoring network health and what kinds of actions are appropriate to automate. There a ton of gotchas in attempting this kind of a solution and they are explored in the How section below.

## How?
> Written with [StackEdit](https://stackedit.io/).
