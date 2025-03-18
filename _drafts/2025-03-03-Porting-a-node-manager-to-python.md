---
layout: post
title: Porting anm node manager to Python
---

After running ANT nodes with antctl and launchpad, I ventured into large clusters of nodes with the community provided tool called [NTracking](https://github.com/safenetforum-community/NTracking) and it's included [aatonnomicc node manager](https://github.com/safenetforum-community/NTracking/tree/main/anm) (anm)

After playing around with anm for a bit, I made a few changes to apply to my own environment and eventually got over 200 nodes running on a test server with a large internet pipe.

Then, something went wrong on my server and all of my nodes stopped. Yikes. And then anm lost visibility into the cluster.

My understanding of anm is that it comes from the days of the early test networks, where it was helping to run lots of nodes that would just be removed en mass when things go wonky or new/test networks.  But in the modern era, I'd like to retain valid nodes even if there is not extra payment for being a long running node right now.

I also want to stop nodes when memory or cpu are high and restart them when things are better without removing and resetting them unless disk needs to be trimmed, the node is shunned or a reset is necessary.

Ultimately, I'd like a way to respond to ongoing metrics rather than one snapshot/minute trying to see what is going on with the server.

## Python

Anm is a collection of bash scripts that do a good job standing up a new server, launching lots of nodes, monitoring them for load on various metrics, performing upgrades of nodes and cleaning up after itself when it is shut down.

I do know bash, but my hopes of are for a script that can also run continuously and react faster to changes.

### Textual

Anm uses [whiptail](https://en.wikibooks.org/wiki/Bash_Shell_Scripting/Whiptail) as a means to create menus and gather input.

Because my plans are for a status dashboard, I narrowed down my options for a TUI to [Textual](https://textual.textualize.io/)  The basic's went ok, but then I ran into problems with light documentation and little code examples to grok from (most links point to the various versions of Textual's documenation).

So... for now, whiptail wins, onto the code!

### systemd service files

When anm goes to start or stop a node, it uses the systemd service option (that currently limits the tool to compatiable systemd linux versions like Ubuntu). Broader compatibility will need to come later, we have enough on our plate to start with.

I start with retrieving a list of all 'antnode*.service' files located in the /etc/systemd/system directory and then reading those files to build a list of possible nodes and their attributes like binary to run, the wallet address, and ports in use.

### Probe the nodes

Now that we have a list of properties for each node, we'll connect to the /metadata endpoint of the potential node to see if it is available. This presumes that the firewall allows localhost connections.

If we can connect to the metadata port, we're able to grab the version and peer id information from the reply.

If we can't establish a connection to the port, fallback by running the node binary and getting the node version that way. Does the node remember it's peerid? Maybe it's derived from the secret key? Something else to figure out later.

### Persist data

I want a way to track when changes to the system have occurred, so I leaned into SQLalchemy for an object oriented solution. The only caveat I've found so far with using sqlite3 as the engine is that sqlite3 doesn't support schemas and all tables in the database belong to the same namespace.

After some fiddling, I was able to load the node configuration from the live machine, save it into the database, and retrieve the information on the next run.

### Migrations

The first real test of this sqlalchemy choice is making a change to the model structure and the underlying database migration.

Database migrations are managed with another package Alembic. Hopefully I can find docs that apply to sqlalchemy 2. Better test before committing to sqlalchemy as a solution.

[Official](https://alembic.sqlalchemy.org/en/latest/front.html) docs seem like a reasonable place to start.

Let's undo my change (making peer_id optional rather than forcing me to use a 0). And initialize the migration system.

Ok, well Alembic doesn't detect any changes in the model, it just provides a mechanism to manage the commit order and we have to write up and down migrations by hand.

This seems well supported, so table it for now and we'll just recreate the database if there are more changes until we start sharing the code with others.

### jsonify models

After adding a Machine class and wiring it up to the ORM, I was testing and discovered that I can't serialize the ORM objects by default (I often use json.dumps to display dictionaries).

The solution I choose (there were a lot of bad options), is to import the json-fix module and create a \_\_json\_\_ method for my classes.

### First module

The model classes are easily imported, so let's shrink our working file by moving these chunks out to models.py.

### Machine Metrics

Now we know the state and configuration of our cluster, lets collect server metrics. This probably should happen first as surveying nodes will cause some load on the system. I'll fix this later.

Along the way, I refactor a little bit to make reusable functions for later. I also decide to add host and default it to 127.0.0.1 when loading configuration from an anm system.  This addition will make it easier to support containers in the future.

I buried a pun when making a variable to report a warning when there are more nodes than a minimum value of disk/node will cross the removal threshold. The whales may have their own solution, so this is for running as a Porpoise. This is just a reported percentage and not acted on currently since the HDRemove will already trim nodes when this happens.

I conspire that when we hit a magic number gb/node that there will be a cascading trimming of nodes causing an inflation of storage and traffic needed as has happened with large shutoffs in the past. Being aware you're playing in risky waters by running with less than 35GB/node is all I can do. My colony is much more modest due to free memory, rather than disk limits.

I plan to support multiple drives since volume groups are fragile like raid 0 by default. Autonomi originates as MAIDSafe, so we don't need to be fault tolerant. However, it's wasteful to loose all our nodes if we can span multiple partitions without lvm to grow with the network.

## Choose actions

This isn't AI (yet?), but I'll model it to support something in the future.

First we need to load our feature matrix, then we'll decide what to do.

This pretty much is a transcription from parts of the anm codebase, with a few tweaks and the suggested addition of Network and Disk IO throttles.

### Remove node

Of highest priority to the cluster is removing nodes when we need to. I have a server with a broken anm installation (anms.sh version was clobbered and caused issues with different settings) that I'm testing on, and it has a bunch of broken nodes, including a few that are buried in the list rather than at the end where anm can detect them.

The low hanging fruit when removing nodes, is to begin with already stopped nodes. Get the youngest stopped node and remove it.  Otherwise, remove the youngest running node.

For safety sake, removing a node will include stopping (in case we are removing a live node) and disabling firewall holes, so those functions get added now. Then remove the node root directory, the service definition file and the log directory.

It takes a while to remove all (over 80) the broken nodes, so temporarily hotwire the logic to ignore that RemoveDelay timer and remove a node on each iteration.

### Upgrade node

There was an updated version of the antnode binary, but this broken server didn't upgrade properly, and there are over 100 nodes to upgrade.

For upgrading nodes, start with the oldest running node. Upgrading starts a node, so take care of stopped nodes with AddNode later.

First replace the node binary, send a signal to restart the node and update the database with the version number and status of UPGRADING.

Fortuitously, moments before I committed the working upgrade code after upgrading my active nodes, a new version of antnode was again released. So this time I get to sit back and let it upgrade while I sleep (and go to work the next day).

### Dead node

I think I discover another broken issue with my nodes, some of the root directories of the nodes appear to be missing. So remove those nodes from the database.

> This turned out be not true, they were just in the wrong storage location because of the clobbered anms.sh earlier. The service definition file correctly had the nodes actual location, I just couldn't match it up when comparing my data volume with the log directory.

### Restart Stopped Nodes

Ok, I had to hold of working on AddNode until the upgrades were finished, adding a node is not allowed while upgrading is underway.

Begin with stopped nodes, there are about 40 nodes that have been stopped for less than 2 days that could be revived.

First check the version number and upgrade if necessary, then add code for start the node, which enables firewall for the node data port and sets the status to RESTARTING.

### Add a node

The last phase here is to create a new node. There is quite a bit to do.

I've been looking forward to the first piece of this section. One of the patches I included to anm (to support more than 1000 nodes) was inspired by a suggestion someone mentioned in the discord (it had been discovered in github), but as provided contained a small bug which broke nodes 008 and 009.  I also have holes from the previously mentioned broken nodes. So, I use sql to find missing numbers in the node record sequence and replace them if found. Otherwise, take the largest node id and add one.

Build a new record requires setting all the variables that make up a node. Then, create the node root directory and the log directory, copy the current antnode binary to the root directory and set them up to be owned by the node operator (ant). Next, create the service definition file and reload systemd configuration.

Finally, create start_systemd_node and enable_firewall with performs those operations and sets the node status to RESTARTING.

With this in place, my missing node numbers fill in and the NodeCap is reached.

## Mysterious Load

Something (currently not determined) is causing significant load occaisionally. And, due to the lasting effect it has on load over 1,5,15 minues, multiple nodes are stopped with a low RemoveDelay timer.

Worse, nodes that get stopped are not starting. It took a lot of time and trial to figure out what was causing the issue:

* Node gets stopped
* Node metadata scan wipes out the version number
* Node gets counted as 'not on current version'
* Upgrade code tries to find a running node to upgrade (not allowed to start new nodes)
* AddNode could not run because there are nodes that need to be upgraded

This required to only update version from metadata if the node responds with one and to add a check for missing version numbers, where the version number gets updated with results of running the binary.

### Adding Stop Counter

Next, when load is high, removal of nodes happens every minute because I solved the counter issue primarily by using status record timestamps so multiples will resolve in correct time. There is currently no 'STOPPING' status because stopped nodes will update as such on the next system probe.

This adds the first true counter (rendered as a timestamp), LastStoppedAt for the last time we intentionally stopped a node.

### Take over cron

I manually disabled the anm cron while testing, but we take over a running anm cluster by removing the old cron and waiting for the existing process to finish (should take less than a minute or two).

This means we want to run in cron. I've been testing with cron running for more than a week, since the RemoveNode code was working, but I haven't had a run-time lock active until now.

### More load

So, in addition to wnm itself being not super efficient in it's db/memory usage (relivant when more lots of nodes are in play), I found that the NTracking system is part of what is causing havoc on my system every 20 minutes. I suspect it's at least the system hit from crawling and grepping all the node data, but it could also be the resulting influxdb insertions and grafana ingestion.  This points to using a centralized NTracking server and splitting the updates to 'overall statistics' like wallet balance, coin price, etc to one script and something else that gathers the node metrics from each server. Perhaps wnm itself can do the reporting since we already gather most of this data for choosing actions. 

