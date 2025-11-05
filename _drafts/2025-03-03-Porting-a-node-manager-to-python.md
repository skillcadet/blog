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

Ok, I had to hold off working on AddNode until the upgrades were finished, adding a node is not allowed while upgrading is underway.

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

So, in addition to wnm itself being not super efficient in it's db/memory usage (relivant when more lots of nodes are in play), I found that the NTracking system maybe part of what is causing havoc on my system every 20 minutes. I suspect it's at least the system hit from crawling and grepping all the node data, but it could also be the resulting influxdb insertions and grafana ingestion.  This points to using a centralized NTracking server and splitting the updates to 'overall statistics' like wallet balance, coin price, etc to one script and something else that gathers the node metrics from each server. Perhaps wnm itself can do the reporting since we already gather most of this data for choosing actions. 

### Over Node count

I went to reduce node count and discovered that wnm only stops extra nodes, where I probably want to remove them if we're changing the node count. So I make two changes, first to trigger remove if TotalNodes is over NodeCap and again in the check for HD Pressure in the Remove path.

The second change is to not trigger Remove if we're in the middle of an upgrade (like we already do Restart), so that the spike in metrics we might get from an upgrade even out before needing to know if removal is necessary.

## Packaging

I now have a deployed version of the code, running on a cron on one of my servers.  There was a little work involved in getting the clone working from the repo. Some of the manual steps are because I wanted to share the colony database between the cron version and the version I'm working on.

To deploy to a new server, there really should be a packaged version that can be deployed with a self installing bash script.  So let's get started.

I know this is a big topic, with heavy opinions, AND there are lot of out dated examples, so I bit the bullet and purchased [Publishing Python Packages](https://pypackages.com/). It's a few years old, but seems to have all the issues I'll need tied together in a cohesive place.

Time to read.  First, there are a few packages to install, setup of some metadata and then a run the first build... Which produced a package with only metadata contents. 

...screech, the current version of the tools do not include my code in the distributions. It seems that the 'upcoming' 2025 updates are not yet published.  So, sigh.

Pyproject.toml were suggested as the future of Python distribution, so I go [here](https://realpython.com/python-pyproject-toml/) which allows me to figure out the current syntax for the configuration file.

Now my built packages actually include the code for my modules.

### Installing

Now that I have a package, let's install it (as editable so I can keep iterating). I begin with calling the program as a python module, which now works.  Then I create what's called an entrypoint that allows me to simply use 'wnm' as a command.

### Pypi

The initial reason I choose the Publishing Python Packages book was it was going to go into distributing the package on Pypi, a package repository for Python.  Luckily the Pyproject documentation goes over pushing my package first to TestPyPi and then to PyPi.

### A new server

Ok, this works on the dev server. How do we setup a fresh installtion on a new machine.

In preparation of this day, back when I first started this codebase, I spun up three little servers and used a default anm installation to spin up nodes to a new wallet. Together, they are not even earning 1 ant/day, but others are reporting no earnings with 200 nodes, so I'm doing better than some. I then discover that two of the three servers do not have firewall ports open for the nodes, so unless they default to relay mode (which I don't think they do), these have not been earning for the last three weeks. Rather than letting all 100 nodes connect to the network for the first time at once, I just reset the servers and let anm run until I have at least three nodes to take over.

I want to install into a virtual python environment, so before I can do anything else, I need to install python3-venv. Then I can initialize and activate the environment so everything that gets installed stays local. I also needed to install python3-dotenv before I could import the dotenv package.

My first installation attempts with TestPyPi fail because I have the package name for dotenv incorrect. This makes me learn how to purge pip installed packages to clear out the broken installation. I also have to remove the broken package from TestPyPi or it keeps getting installed along with the current (also broken) package.

My current gatcha, no one created a json-fix package in TestPyPi.  I don't think I want to go through the headache of publishing json-fix myself, so I push the current version of wnm to the live PyPi site. Now I can confirm that installation succeeds!

Oops, I've never tested the code to remove the anm cron. But I also figure out how to use the Live PyPi site as a fallback when using TestPyPi as a primary source, so I can stop iterating on the PyPi site.

Ok, wnm properly disabled anm and ingested the cluster. Create a cron entry and, voila, we have a new wnm node.

### black

One of the tools recommended for my project was [black](https://github.com/psf/black), a code formatter.  I hope this to be a long lived project, so I process my modules to check how inconsistently I've done.

I know that I dislike Python mostly because whitespace affects the execution of your code and I prefer to use whitespace more liberally than allowed.  I also tend to use single quotes for constant strings as other languages I have used have better performance when using single quoted strings.

I don't like the way black treats comprehensions or how black ends the indentation on multiline expressions.  There are a couple sections that are harder to follow being expanded by black, but otherwise I'm onboard with the changes.

### isort

The final tool recommended, [isort](https://pycqa.github.io/isort/), this simply reorders the import packages sections of code files, splitting the imports into groups by category and then alphabetically. `isort` also supports `black` formatting, so it splits long lists of imports into a multiline list.

### src

I had followed other advice I can't locate now, that recommended putting your modules in a module directory instead of a `src` directory, but I now think that ambiguously meant `src/module` instead of just `module`. Let's see what happens if we just push those all down a level (so we can separate out `test` paths later). This should require updating the path to search in the pyproject.toml file.  There was one code change (beyond the reformatting that happened with black/isort and moving the paths), so we'll update the version number.  Building the package, I still see the source path in the distribution tar.

Pushing to TestPyPi, I can test doing an upgrade with pip install of my package and things look like they are working. Just to be on the safe side, I clear the pip cache and uninstall wnm before installing again and confirming things are working.

## Configuration

Ok. I've put this off as long as necessary. Creating a new cluster from scratch means we need to collect a bunch of information from the user.  But before that, we have a migrated cluster to manage with the same options.

Right now, the only way to make changes to a live cluster are to update the sqlite database with new values. Obviously, that is not ideal.

The lowest hanging fruit is what already exists with dotenv. I can use the .env file, and/or I can create a regular file that contains the settings to update.

Start with a new function update_machine() that will go over all the settings, see if a settings is defined, verify that the value is the same or the function will update the settings before making choices about the server.

### ConfigArgParse

There were lots of complex options to getting configuration working (various file formats, modules, etc), but I choose to go with [configargparse](https://github.com/bw2/ConfigArgParse) to have fine control over precedence.

This turned out to be quite verbose codewise, but I can now derive configuration from defaults, config files, environment variables, or command line options (in that order).

### Merge

I built the config module in isolation, then began the long slog of making the spike version of wnm integrate with the config module.

This took weeks and involved refactoring to a utils.py and common.py.

### Upgrade degrading performance

Around this time there was a new release (3.0.9) of the antnode binary, so I updated the server's base location and let the system roll those changes to the colony.

Then things started to go awry. The new version uses 30% more resources than that previous versions, causing the system to go over it's resource limit and stop, but not remove nodes because the system is not having disk pressure.

The first step was to lower the nodecap to get to a reasonable amount of nodes. This required dropping from 200 to 130.  This allowed the remaining nodes to upgrade.

Later, I changed the remove node logic to trim nodes instead of stop them if we're hitting spikes during an upgrade. And then made that less aggressive by only trimming more nodes if there are no RemoveNode events running. (Disk pressure and NodeCap limits are still aggressive)

## Claude - October 2025

I toyed around a little with Claude in a new project to learn something on how it works. Let's try to step our speed up here with a little cash.

Oops. Things went fast with Claude, and I didn't keep this blog up to date while it was happening.

I may come back some day and rehash the tale using the commit logs to remember the timeline.

I started with building a CLAUDE.md in the code repo and then importing and analyize the code base. I then instructed a few changes I wanted to make and built a plan that incorporated some suggestions from Cluade's review.

### Tests

Ok, first up, generate tests so we know if something breaks.  This is a little complicated, because I'm on a mac and I don't yet support OSX as a node management environment. I did discuss that as a target, but that I want to start by getting tests to work in a linux docker container before getting the project to work on a mac.

This is a little tricky as this won't actually start a node, since there is no process manager inside a docker container, but we can test the python functionality otherwise.

### snake_case

Apparently, the recommended format for parameter names uses snake_case instead of UpperCamelCase. This seems like a GREAT task to automate with Cluade.

I am unlikely to trust AI to do things on it's own, so I validate all the changes Claude recommends and often request or make alterations to the presented solutions.

### The future

In addition to OSX support, I planned for: making multiple operations per cycle, converting timers to seconds instead of minutes, Docker containers as node environments, 

### Abstract out Process Managers

Ok, the initial implementation brought in systemd, running as the 'ant' user, with passwordless sudo access.  But I want to support managing nodes that run under multiple options and environments.  To give Claude some idea about how to split things up, I planned for multiple options.

1. Linux Root systemd
2. Linux non-root systemd
3. Docker
4. Setsid
5. OSX
6. antctl
7. forimcaio
8. Dimitar

I start implementation with Linux root, docker and setsid to get a base for how to build the rest.

### Abstact firewall

OSX and docker don't use 'ufw', so I need to abstract this out and provide a 'null' option.

### Decision Engine

To be able to run concurrent actions, I need to seperate the choose_action function into a planning stage (decision_engine) and execution logic.

### OSX support

Ok, this is enough to start getting things to work locally. First, I need to setup a Python virtual environment, which means installing pyenv, which means installing Brew. Luckily, I already have XCode installed (I got that out of the way on the first day of this laptop).

Beginning with a plan, the first items on the agenda are system uptime results. As with many things, OSX has a *BSD foundation, NOT a linux one. This means there are differences in names and results for certain operations.  System uptime is used to calculate if the system has rebooted since the last time wnm was run.

Another change is how we get the cpu count. In linux, I am using a kernel specific result that gets the number of CPU's assigned to this session instead of the total number on the system. This doesn't apply to OSX, so we use the generic os.cpu_count method.

The paths that are appropriate for OSX storage are annoying for containing spaces in the paths, in addition to being long. Sine these were mostly hard coded before, this becomes a configuration method that returns the correct paths based on the detected operation environment.

Finally, I can add a new Process Manager for `launchd`, what OSX uses for managing services.

#### Refactor utils

The executor has been calling the utils.py methods that were inherited from the initial port of `anm`. Most of these operations actually belong to the specific process managers instead.

#### Pull database changes to the executor.

This separates the process managers from doing database updates directly. A win. (Since I'm writing these updates from the future, I already know this becomes a new challenge.)

#### OSX tests

The OSX specific tests are not numerous, however, we want them to increase coverage.

### Node ID selector

In my prototype, I had selected a novel approach to choosing the next node id, by scanning the database for holes from removed nodes. But, in my testing I found a bug. My code starts from the first node and searches through to the last node. The SQL fails to notice if the first node doesn't start at 1. This is a quick fix to make sure there is a node 1, otherwise, return the first hole or the last node+1.

### forced_action

Ok, getting started on the road to multifunctionality...

First up, I need a force_action command so signals can be sent by invoking wnm directly. I add all the main actions and a new 'teardown' action that will reset an entire cluster in one call. In theory, this should require a --force argument since a TUI isn't planned to get interactive input, but teardown is kind of already a forced operation. Something to ponder.

Now, I want to be able to specify specific nodes for operations, so I add --service_name as an optional parameter, we default to the built in logic over which node to select if --service_name is not defined.

Next, a new foced_action, 'disable', to finally allow that state (which has been tested for since very early on in the project). Disabling a node turns it off and prevents it from being scanned during a survey. 'start'ing a disabled node with forced_action is allowed as that bring a node out of DISABLED state.  Perhaps this also needs a --force argument.

## Reports

Ok, we can make changes to the system, it's time to starting seeing output from the environment.

Create a --report argument that selects which report to run. Then add a new action to force_action called 'survey' that will run a site survey before running the report. The default behavior is to NOT rerun the survey as on large deployments, that can take significant resources.

The first reports. 'node-status' and 'node-status-details', mimic the output from the 'antctl' reports.  Then I add --report_format option to allow for 'json' output vs the default 'text' output.




