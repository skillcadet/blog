---
layout: post
title: Porting anm node manager to Python
date: 2025-03-03 12:00:00 +0000
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

Then things started to go awry. The new version uses 30% more resources than the previous versions, causing the system to go over it's resource limit and stop, but not remove nodes because the system is not having disk pressure.

The first step was to lower the nodecap to get to a reasonable amount of nodes. This required dropping from 200 to 130.  This allowed the remaining nodes to upgrade.

Later, I changed the remove node logic to trim nodes instead of stop them if we're hitting spikes during an upgrade. And then made that less aggressive by only trimming more nodes if there are no RemoveNode events running. (Disk pressure and NodeCap limits are still aggressive)

## Claude - October 2025

I toyed around a little with Claude in a new project to learn something on how it works. Let's try to step our speed up here with a little cash.

Oops. Things went fast with Claude, and I didn't keep this blog up to date while it was happening.

~~I may come back some day and rehash the tale using the commit logs to remember the timeline.~~

I started with building a CLAUDE.md in the code repo and then importing and analyize the code base. I then described a few changes I wanted to make and built a plan that incorporated some suggestions from Cluade's review.

### Tests

Ok, first up, generate tests so we know if something breaks.  This is a little complicated, because I'm on a mac and I don't yet support OSX as a node management environment. I did discuss that as a target with Claude, but that I want to start by getting tests to work in a linux docker container before getting the project to work on a mac.

This is a little tricky as this won't actually start a node, since there is no process manager inside a docker container, but we can test the python functionality otherwise.

### snake_case

Apparently, the recommended format for Python parameter names uses snake_case instead of UpperCamelCase we inherited from 'anm'. This seems like a GREAT task to automate with Cluade.

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
6. antctl - the official Autonomi node management tool
7. forimcaio - another Autonomi node manager that uses containers under Docker or Podman to manage nodes
8. Dimitar - an Autonomi user who deploys 200 node per Docker container for a total of 6k nodes/server

I start implementation with Linux root, Docker and setsid to get a base for how to build the rest.

### Abstract firewall

OSX and docker don't use 'ufw', so I need to abstract this out and provide a 'null' option. Then I add code to choose the firewall option based on the environment we're running in. This will need to allow us to override which firewall to use in the future, for example, if we're running as root on a linux server that doesn't have a software firewall like 'ufw'. Being able to specify 'null' or a new firewall manager gives more flexibility.

### Decision Engine

To be able to run concurrent actions, I need to seperate the choose_action function into a planning stage (decision_engine) and execution logic.  This is a big refactor, but that's why I started with tests.  Claude makes this a lot simpler than I thought it would be.

### OSX support

Ok, this is enough to start getting things to work locally. First, I need to setup a Python virtual environment, which means installing pyenv, which means installing Brew. Luckily, I already have XCode installed (I got that out of the way on the first day of this laptop).

Beginning with a plan, the first items on the agenda are system uptime results. As with many things, OSX has a *BSD foundation, NOT a linux one. This means there are differences in names and results for certain operations.  System uptime is used to calculate if the system has rebooted since the last time wnm was run.

Another change is how we get the cpu count. In linux, I am using a kernel specific result that gets the number of CPU's assigned to this session instead of the total number on the system. This doesn't apply to OSX, so we use the generic os.cpu_count method.

The paths that are appropriate for OSX storage are annoying for containing spaces in the paths, in addition to being long. Since these were mostly hard coded before, this becomes a configuration method that returns the correct paths based on the detected operation environment.

Finally, I can add a new Process Manager for `launchd`, which OSX uses for managing services.

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

### Reports

Ok, we can make changes to the system, it's time to starting seeing output from the environment.

Create a --report argument that selects which report to run. Then add a new action to force_action called 'survey' that will run a site survey before running the report. The default behavior is to NOT rerun the survey as on large deployments, that can take significant resources.

The first reports, 'node-status' and 'node-status-details', mimic the output from the 'antctl' reports.  Then I add --report_format option to allow for 'json' output vs the default 'text' output.

### multiple node specification

Now, as a step towards multi-action, I add comma-separated node name support for --service_name, to limit the output of the report to just the node(s) that are requested.

Then I extend the comma-separated node for --service_name to the other node actions (except for 'add node' which doesn't support named services).

Next, I want to allow multiple actions on unnamed services with the addition of --count parameter that supports 'add' in addition to 'remove', 'start', 'stop', and 'upgrade'. These actions will choose the affected nodes. For example, remove and stop affect the youngest nodes while start and upgrade affect the oldest nodes.

## Release v0.0.11/v0.0.12

Ok, I have something worth iterating on other systems. To make that easier (I've been testing by rsyncing my updates to a linux server under a virtual environment), it's time to package up the app and push it to Github and PyPi.  I had previously deployed 'wnm' as an alpha package, both on TestPypi and Pypi, so this just entails updating the version number, building the package and pushing the deploy files. Easy peasy.  I take the additional step of making a Github release as well, which is the first time this project has been tagged.

### multiple weighted rewards_address

Ok, testing on systems that I can't cut and paste on means I have to type a 40 character string for the required rewards_address parameter, even though I personally will default to the community faucet wallet.

To resolve this, I add support for two named wallets: 'faucet' which will use the hardcoded wallet address in the code and 'donate', which defaults to the faucet address but IS changeable via command line options.

I figure it's important to distinguish between the two, because the next step is to add support for multiple, comma-separated wallet addresses. Then, I extend that even further by optionally adding colon-separated weight value to affect the distribution of the wallets across the cluster.  eg: "--rewards_address 0x40CHAR:100,donate:10,faucet:1"

### CI/CD - Github Actions

Along the way, I added configuration for Github Actions to run my tests when I push code to Github. Claude doesn't get that right on the first attempt, but iterating slightly gets the builds working.

This then inspired me to getting the last failing tests to pass, so I could see a green status online. 

### Documentation?

Hmm. There is a lot to document and it's only going to grow geometrically worse. Actionable reference guide material may be spread through what's been captured in code, comments and artifacts.

A lot of this is going to be repetitive descriptions of similar features. Claude can use my templates across them all. 

Well. I generated a high level outline with Claude and then I started reading the existing documentation and found some bad assumptions that means we broke the original ProcessManager, so...

### systemd+sudo

First, I confusingly called this linux+root mode when it's actually running as a user and controlling the system with passwordless sudo.

I have a ubuntu desktop running under parallels to test on. This requires passwordless ssh and sudo for Claude.

It doesn't take a lot to get the systemd+user working again, but how about with sudo. This is for running with more resources than a regular user can use or if user processes stop when the user logs out.

Ugh, apple is forcing me to reboot tonight so I'm going to be abandoning an active seesion (I ran out of tokens and). I really can't stand not being able to postpone upgrades that will reboot. F* you apple. At least tonight I noticed it was going to happen before I lose more of my work.

Ok, the error was in the code that defined the service file paths only defaulting to systemd+user and not reading the configuration item from the database.

Which brings up the database. Part of the problem for Claude is that the runtime is defaulting the db location to ~/colony.db instead of BASE_DIR.

This requires some refactoring and code duplication to allow searching for the database in specific locations based on platform if it's not defined with --dbpath.  Which discovers that we lost the ability to define dbpath somewhere along the way.

### version and remove_lockfile

I added some convenience features.

--version will skip most of the system bootstrap and simply output the version and exit.

--remove_lockfile will remove the wnm_active lockfile and exit.

### Documentation take-2

Ok, I flesh out the begining of a user guide (Part 1). I didn't log as many notes while I was developing this, I'll be more diligent about keeping the docs and code current.

So, again, reviewing the output I find lots of corrections, including some code changes like misnaming Launchd manager as Launchctl (a command launchd uses). Luckily that was mostly just documentation changes.

The suggested commands to add to cron don't work and need to be run (generally) inside a virtual environment.  I also need to fix (again) the github url and pypi package names, maybe I didn't save some of my changes and hit accept instead.

Then I generate a functional reference (Part 3)

### Refactor teardown

Teardown was originally going to be a command line argument option, and I had built part that required --confirm flag to also be set, but before I built a backend to it, I added teardown as a --force_action option.  So the correctly formatted one is a stub and the working one has no protection.  Fixed

## multi-node containers

There exist deployments of thousands of nodes that run using the Autonomi offical node running tool 'antctl' inside docker containers. And adding resource management around antctl was an original inspiration of creating wnm in the first place.

Docker and Antctl process_managers have been on hold until now because they both add a layer of abstraction to our deployment.

Docker and S6-Overlay nodes all run inside of a container_id that is not predefinable.

Similarly, antctl maintains it's own node naming scheme that is predictable but immutable.

Based on my decision to separate the process managers from direct manipulation of the database, we have to: collect, return, and update the database with these return values from the add_node process to the executor.  Luckily, I already have a type defined for a NodeProcess, so it's just adding a new field and passing the object back. Small but crucial to getting these new node managers online.

I feel a larger use case for antctl directly over running antctl inside a container at this stage. People want an easy way to use antctl. I'll get to s6 later because it'll be fun to contribute to that environment.

### antctl

To get claude up to speed, I start by taking of a copy of the antctl README, strip it down to parts I'm using, then extending that further with supplementary information on getting the node details from antctl.

Then I generate and iterate on an implementation plan before begining the task list.

After coding the initial methods, I start to integrate antctl on my mac. I manually add a node to my antctl environment and then work through the issues until my --init request successfully loads the preconfigured nodes. 

### --no-upnp

Upto now, I've been hard-coding the --no-upnp flag (added in May 2025 and broke my nodes), but clearly that won't work for a deployment that needs upnp. So I quickly wire up a new argument to disable upnp and then update all the process managers to respect that setting.

### NTracking and influx

The `anm` script this project is based on is part of a larger package called [NTracking](https://github.com/safenetforum-community/NTracking) which is a platform for monitoring nodes.

I had built a shim script called UpdateNodeDetails.sh that replaces /var/antctl/NodeDetails (a shell array) with a call to the database to generate the node list. However, this only works for installations using /var/antctl/services and don't contain any missing nodes.

So, I add a new report 'influx-services' that will output the same data as the NTracking script.

This entails adding a `-q/--quiet` flag to not emit system metrics during a job run.

Most of the output from the environment correctly uses the logging package for output, but quiet mode means I have to remove the remaining print statements.

I don't want to get telegraf working on osx, much less the whole NTracking stack, so I inefficiently install everything running inside a virtual machine on my mac. Hey, that's why I got a laptop with so much memory.  This then gets documentation on how to send the influx-services report via file output on linux, or submitted over ssh from osx from cron.

### Clobber bug

After some debugging, I discovered that --no_upnp is being clobbered when it's not specified on the command line. That's a problem. There is some logic mismash between a true state for a negative option that has no way of reversing the operation.

### More antctl shenanigans

Antctl adds a node during create but doesn't automatically start it. That's easy enough to fix in wnm. Before we can start it, we needed to be able to capture the service_name made during the add node phase.

Antctl also handles binaries in a particular way. If --path is not specified, antctl attempts to download the binary every time it adds or upgrades a node. To get around this one, store the current antnode path in the machine_config, make it configurable, and use it where we deploy code.

### Database migrations

Ok, Alembic was integrated early on, but there isn't a mechanism to detect and run needed migrations.

First, store the version number in the database. Then, when the schema's don't match, give a warning with the description on how to run migrations (with the new --force_action wnm-db-migrations --confirm) and exit.

### Cluster rebuild

So, there is already logic to import an anm generated cluster or an existing antctl managed cluster. I extend this capability so that we can import existing nodes from the other process managers in case we clobber our database and need to rebuild it.

But this causes a problem during --init, where on a fresh installation we generate warnings that there are no nodes.  To solve this, add a new --import flag for --init operations that only triggers a survey when --import or --migrate-anm are present.

### Clobbered logging

Something that was changed broke logging. Info or debug level logs show no output.

After timing out with two sessions of claude, I found the bug. Alembic was resetting loglevel to WARNING regardless of what we set on the command line.  Tsk tsk.

### More reports

I want to make the default run quieter by default, but keep the option to emit data without resorting to DEBUG logging.  I solve this by creating new --show_machine_metrics, --show_machine_config, --show_decisions which emits these outputs during a regular command run.

Then, I add two new reports. 'machine-config' and 'machine-metrics' to be able to retrieve those reports in text or json formats.

### Rate limiting survey

In theory, running a survey on a large cluster could cause visible load on a server. To allow for spreading things out, I add a --survey_delay feature for milliseconds of delay between each node. Getting a hang of adding migrations right away

### Test debt

Getting some test errors, this resulted from tests being out of sync with the recent database schema changes. A quick fix.

### Migration bug

Running a check on making sure the database migrations are all correct, I discover that somehow the new --survey_delay got added with a separate HEAD in Alembic.  Since nothing else has changed, this is an easy to fix the migration chain.

## Official Concurrent Operations

Ok, concurrency finally arrives by fleshing out the --max_concurrent_[start|upgrade|remove] settings with cli arguments, along with a new --max_concurrent_operations that is a global limit on total actions.

It get's a little more involved as we can have fewer of a state than quotas allow. For example, the decision to add 4 nodes when there are 2 stopped nodes and the node cap allows more nodes would start 2 nodes and add 2 nodes.

Now the cluster can aggressively scale up and down when configured properly.

### Update_config as a no-op

I added update_config/noop options to --force_action to give a targeted way to update config settings without running the decision engine automatically.  Before, there was a way to make changes by requesting a --report, otherwise the decision engine runs after settings are resolved.

### More migration headaches

I think I have figured out the cause of my migration system still not working. Apparently there was a race condition in the auto tagger that was causing the alembic_version table to not be created. This mean no databases currently have their version number (well, now with v0.3.3, but none in the wild) so can't be migrated without novel forensics that a simple rebuild of the cluster will solve.

So, to test this I need to have something to migrate too. so I add ...

### action_delay,this_action_delay/interval

Now that we do things at speeds less than a minute apart, we might need to limit velocity of a set of operations.

--action_delay milliseconds is a persistent setting that adds a delay between repeated operations.

--this_action_delay milliseconds or it's antctl alias --interval milliseconds is a setting that does not change the machine configuration, only applying the time delay to this execution.

### ENV report format

I add an env report format to the machine_config report. This is to allow a user to backup the machine config settings (not the node details, which have their own reports) into a file that can be reloaded later as a config file during init or operations.

Which produced an error. Before, attempting to set three immutable settings (port_start, metrics_port_start,proces_manager) would result in an error except during system --init.

This caused a failure to import a reported config as those settings were not allowed to be present.  So, I changed the behaviour to only give a warning if the values are different than what is saved in the database.

### this_survey_delay

I had the idea for this feature before this_action_delay, but action_delay was added first because I needed something that changed the database so I could confirm that migrations were working correctly.  This allows us to set a custom, non-persistent time delay between node checks when running a survey.

### --json shortcut

We do have --report_format json already for some reports, but I added a shortcut --json so that (like --interval) we support similar settings to antctl.

### ENV formats

I decided to add --report_format env to the machine-config and machine-metrics reports. This allows exporting those values back to the shell to be used in other tooling.

### Confirgparse capitalization

We found an issue... configparse, when loading a from configuration file, uses the argument name (node_cap) instead of af the environment variable (NODE_CAP).  This requires fixing the documentation, but also implies the need of a 'config' --report_format so we can export our settings for later.

### --init UX

So, it's kinda bugged me for a while, that after an --init, wnm auto detects the system as having been rebooted.  I fix this by adding logic that detects when doing an --init, to skip the decision engine, set the start time as current, and report that the system was initialized and exit.

I also changed the --init phase to NOT run the decision engine on a newly provisioned system.

Next, the migration autotagging system creates a blank database when we start wnm on a system that hasn't been --init'd. That is a simple fix, to check for the database existing before attempting to connect and only allowing it to not exist during --init.

### Port start ranges

Currently, as inherited from anm, the port_start, metrics_port_start and rpc_port_start ranges are the thousands digits, so port_start=55 means 55000.  

It's natural to put 55000 in for this value in the configs, so extend the evaluation logic to resolve the port range in both cases.

Note, this solution currently rounds down to thousands, so it's not possible to start ports at 55555, which would truncate to 55000, or the first usable digit 55001.

Which reveals that we're still hardcoding the start ranges for metrics and rpc ports.  A small change is all it takes to get things working properly.

### lock file shenanigans

Something is causing the lock file to be left behind. So, implement signal handling and atexit handling to provide a safer way to cleanup the lock file.

### add debug log

When loglevel is set to DEBUG, output the decision engine, machine config and machine metrics.

### Virtual environment tweaking

The documentation needs tweaking on usage of virtual environments on different platforms. This is an iteration and not a final result as there are more platforms to test.

### Focus on user modes

root/sudo operations are not a standard implmentation, so move that example from the README to the end of the user guide part 3.

### antctl path

So, OSX cron does not pass the PATH environment in a way that subprocess can find the executable. So add --anctl_path to allow a full path specification.

### remove antctl suggestions

The guides were installing antctl for all process managers, which is unnecessary, so removed those lines from the docs.

### antctl debug

antctl has a --debug parameter that provides more output from the commands, so I add --antctl_debug as a flag which will activate the parameter when when present. This stays persistent when activated unless manually tweaked from the command line.
`sqlite3 ~/.local/share/autonomi/colony.db "UPDATE machine SET antctl_debug = 0 WHERE id = 1"`

### Antctl+Zen mode

There is currently an issue happening, where antctl won't start a node and we get stuck. There was some strange path's happening due to wnm being opinionated about where things go and antctl which wants to use different paths.  The zen version of the antctl manager does not set any paths and lets antctl decide where the node parts go. The output of the command is processed and the settings are updated in the database. Port numbers are still set so they can land within defined port ranges. Could we allow zen mode to choose upnp ports also?

### antctl --version

antctl has --version argument, that when used with add/upgrade sets the version number to use. wnm already has specific behaviour for --version, so added `--antctl_version 0.0.0` which will pass the version number to antctl.

### RUST_BACKTRACE

Still unable to figure out why certain antctl managed nodes are just quietly dying. Adding the ability to pass in RUST_BACKTRACE environment variable to hopefully get help from autonomi.

### antctl removed nodes

antctl remembers removed nodes and will not allow a new node to use the same port number until a `antctl reset` is performed. This conflicts with wnm default behavior of filling in holes to stop flapping servers from running out of ports. wnm's current behaviour is to STOP a node instead of remove it if memory/cpu/io are spiking, so hopefully this doesn't become an issue.

This requires a new method to track the highest used node_id and always use a new one when adding.

### repr/json out of sync

The internal \_\_repr\_\_ output was missing colums, as were my custom \_\_json\_\_ methods. Cleaning that up will help with debugging.

### disable_config

There are currently two flags that only persist a true setting, --antctl_debug and --no-upnp and ignores not being defined as a change of state. This causes an inability to turn off the setting without restorting to directly editing the sqlite3 table.

The workaround for this is a new 'disable_config' action which inverts the logic and if either of those flags are set, the database is set to False.

