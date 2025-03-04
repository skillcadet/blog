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

Because my plans are for a status dashboard, I narrowed down my options for a TUI to [Textual](https://textual.textualize.io/)  The basic's went ok, but when I ran into problems with light documentation and little code examples to grok from (most links point to the various versions of Textual's documenation).

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
