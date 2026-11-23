---
layout: post
title: Home Alone Program
---

My mother lives in an ederly community, which has established a Home Alone program where vulnerable residents check-in daily. She asked me if there was a way I could provide backup help in case something happens to the current organizer.

I meet with the gentleman over the phone and remote desktop, where he shows me the tool he's using, how to add/remove a unit, how to deploy changes to live, and where checkins get logged.

The first thing we identified was a need to share the site, with the bonus of backing it up as well. I walked him through creating a github account, installing github, finding where his app stored data, creating a private repo, and pushing his repo up.  After adding me as a contributor, I was able to download the site. Make a change, commit and push it, then show him how he should do a git pull before making changes, committing and pushing after he's done so I can make changes if I need to. That's one bus avoided.

The program is run by one main person, and a few volunteers who take turns doing rounds after the cutoff time.  Some houses check-in themselves using a web form. Others place day of the week signs in their windows to show the volunteer they are ok.

By accounts, it takes about an hour a day to check through the emails and note who has/not checked in and follow up.

I can't volunteer to be available during that time window because of work, so I wanted to come up with a more sustainable process.

What I proposed is a map interface that instantly shows the current status of all the units that are participating.

I then learned that the (very simple) webform submits to a third party service that records the entry and forwards an email to the programs email address.

This form is hosted on a micro ($3/month) server running cpanel and not appropriate for a nodejs/python service.

So I decided to go with a modern svelte 5 frontend against a php rest api backend so it can continue to use the same host.

It took a bit of trial and error to get credentials for the vps, which doesn't provide ssh (expected at this price range) but does do tls ftp. 

I then asked my mom to have the staff send me a high resolution map of the park.

## Map Editor

Most of this solution revolves around the map and it's parcels, so rather than faking a frontend to build the backend against, I started with the map editor.

I've been pairing with claude for about a year now, off and on, on a couple of learning projects, so I decide to try a more tightly integrated workflow. Including auto mode for a try since I'm still learning svelte and I've forgotten php (I knew it back in the procedurate days and didn't use Object oriented skills there).

I'm getting better at prompting for Claude because the first iteration of the map editor was spot on, including a way to not only magic want detect a unit, but to run a detect unit across the whole map at once. I didn't specifically ask that because I didn't think that would work in-browser, but I was a little vague and claude suggested a solution. The first change I had to make was that I asked for zoom buttons and navigation sliders for the map, but claude overlayed the vertical slider over the zoom in button and I have to fix that.

For Autodetect/magicwand, I didn't attempt to ocr the lot labels because the detail wasn't quite good enough and the labels are in all different orientations. I did add a visible difference between unlabeled and labeled parcels so it's easy to spot units that are not yet labelled.

It was a very mouse movement intensive to edit a map:
* click a unit
* move mouse to parcel label entry field
* enter unit number
* mouse over to next property

Instead I improved the UI to autofocus on the label entry field after clicking a unit. This made clicking between units very fast. click/type/nudge.

When selected, a parcel shows it's boundary nodes. I didn't include a way to delete nodes, so when the magic wand can't properly detect the boundaries (most often because the parcel text touches the boundary) and overly complicates the shape, instead of keep attempting the autodraw with different boundary levels, it's faster to delete and re-draw the new parcel.

Ever continuing on the tedious interface use, I added hot keys if a node is not selected, 'd' goes to draw mode (allowing to start a new parcel quickly if you have to hand draw many). if no unit is selected and the 'm' key is pressed, goto magic wand mode.

Next I added delete key to delete a selected property. But then I ran into a problem. Apparently, OSX defines the delete key to a backspace, so hitting backspace when editing a label was deleting the whole parcel.

My solution was to only delete a property with backspace if it's label is already empty. still liable for accidental parcel deletion, maybe if other OS properly are detected as delete, instead of just prompting for delete on an empty backspace, it can prompt with delete and the osx keystroke (Control+D or Fn+Delete).  Something to add to the todo rather than chase now since I'm the only user of editor.

I built the map editor to use localstorage to solve the api chicken/egg issue with not having a backend for a frontend that doesn't exist yet.  This allows me to not need the server taking lots of little actions.  Instead there is a sync to backend option along with the import/export locally features.

## Backend

Ok, I don't want to deploy anything to the production server yet and I want to iterate locally, so I use vite for a development frontend ui in a docker provided LASP stack (LAMP with sqlite3 instead of mysql) container to provide php backend.

First up, we need a schema that supports users, devices, roles, parcels and sites/parks.

Then we need a way to start a site. I'm using a web of trust system instead of a user/password/email auth system, so the site needs a recovery key in case the site administrator become unavailable.  We also need a way to receive the map editor synchronization.

When a new install is discovered, confirm the user has copied the recovery key and then give them access as a site admin.  I had to tweak the contrast ratio for the recovery token, it defaulted to very low. I also removed 0/O/8/Q/G and I/l/1 from generated codes since those are harder to distinguish.

### moving quick

more of Phase II: create checkin and away APIs based on the existing webform. I also add a notes system. Finally an away privacy setting, that shows you as pending before cutoff and checked in after cutoff. This includes setting a cutoff time that reflects on the check-in form text and the live view color triggers.

Phase III is the web of trust authentication mechanism. 

The site administrator is a role, a user can have a unit associated with their device (only these users see the checkin form). Users can invite more devices to their account. helpers/site admin and staff (non-exclusive roles) see the live map view which shows ONLY participating units and their status. This view allows them to see the status and notes and also to check-in for a user. The staff role can activate and deactivate units from the program, set notes, generate unit invites. and do checkins. Then I add auditing for who does actions.

### transition

there are about 80 active units in this park, in an effort to streamline the onboarding of new tenants I build a self-registration form that looks identical to the current checkin screen (select a unit and hit checkin, removing the need to select todays date) that automatically registers that unit to the device, without needing an invitation code.

A little security interruption, I can see directories when browsing the site, so use some .htaccess rules to make sure exposed directories get blocked if they end up in the public folder structure.

Now this moved really fast, we kinda skipped testing. Fixing that creates 33 tests.

Wire the frontend svelte to the backend php rest api so things actually happen.

Once the site is live, an unlogged in view is a self-registration screen that asks for an invite code, a name and a device name.

Allow invite codes to be entered without the dash in the middle.

Now real data can happen, but that collides with testing, so configure docker to use temporal containers instead of clobbering the development database.

To be even safer, when tests contain a X-Test-Mode header, don't even attempt to use the production database name.

### Bulk tool

I've not been idle while Claude thinks. Beyond the autodetect parcel fixes, I still had to hand label over 500 units. Looking at the map, I don't see much logic between how the units are laid out on the map. Thus activating the rest of the 80 odd active units would be tedious for me who is not familiar with the layout. So I create a bulk (whitespace separated) tool that will activate or deactivate a list I retrieved from the active units. This currently lives in the map editor so only site admin can see it. I'm considering allowing staff access to bulk activate/deactivate.

This leads into another issue, the live view renders ALL lots and marks them as expired. This requires a lot of transfers for parcels that are not a helpers concern, so for live view, only send the lots that are active and just hide deactivated ones.

### device testing

ok. now that we have a functional prototype, I want to see how a mobile device works. This entails binding Vite to the network so I can view the development service on my laptop from my phone.

Extending the invitation system to allow additional devices for the same account lets a user self enroll. Staff/site admin can invite a prescribed unit and helpers. Site admin can review and revoke invitations or devices.

Extending invites even further, I now generate QRcodes that deep link the invite. And then a redeem form for someone who is not authenticated.

Make the live view only visible to helpers/staff/site admin, do not complicate the tenant's interface with potential private data (helpers are a known compromise).

Allow more than one invite code per user so they can register mulitple devices/browsers.

Switching from localhost to a private lan ip involved an invite for the new domain. This required a domain/prefix setting for the invite links/QR.

## Check-in

All of this builds to the actual point here, checking in. The legacy form had users select their unit, select the day of the week and submit.  TMHAP can be more powerful than that.

The Web-of-trust mechanism means the system knows the unit a tenant belongs to, and what date/time it is, so all the user needs to do is load the form and hit submit.

I also added the link to the dark/away panel.

Then, once they were working, I copied the original layout, theme, and branding of the legacy site to the transition endpoints to make this as similar as possible to the existing users.

I also created the ability for a check-in action to be re-broadcast to an auditable offsite location. This just mimics the current webform behavior that creates a database entry and an email (currently used for manual coorelation of who has/not checked in)

### Collaborating with the park

Ok, now the site is an mvp with the basic parts started. Let's deploy it so the park organizer can see what I've done.

I don't want to take over the live page yet, which requires updating my code to support being run from a sub-path. Not difficult, but Claude had made assumptions that needed to be fixed.

Next up was addressing how to get the files up to the server. After considering my options, I chose ftps (secure file transfer protocol) and built a reusable deploy script that stored my credentials in my mac secure keychain and not in the file system.

### Secondary Site Admin

Really, I'm the secondary site admin. And in a soft-handoff from one site admin to another, you invite the second admin (you could generate multiple of these as backups) who revokes the old admin's access. To grant access to the park organizer, I need a role specific invite. 

Sharing this invite code (10 characters with an optional hyphen) ended up being easier by adding a copyable full URL deeplink with the invitation redemption code filled in.

I also added the ability to add a role invite to an existing account so there are not lots of accounts per user.

### Feedback!

Ok, the first actual request was changing the checked-in status from blue to green. I am selecting from a color-blind safe palette so I don't really have to think about which green to use (a kind of blueish green).

Also added, when a unit is searched for, it disaplays a pulsing border on the map so it's easier to see. Then, restrict the map editor to the site owner.

### Staff interface

Activating and deactivating units from the map is a 'staff' role. This adds a new tab 'Manage Units' that shows the full map (all defined units) of the site.

The color coding of this page is: dark gray for deactivated units and blue for activated units. Staff can see the live map, so that is not duplicated here. When selecting a unit, the interface allows to activate/deactivate a unit. Add/see notes and generate a unit registered invite code/link/qrcode.

Then add the staff role invite for the site admin people and devices page.

### Legacy registration

The organizer wasn't sure if self-regisration was a good idea and asked it to be disabled. Rather than let this one decision define the product, I added a UI element that enables an automatically closing window (default 14 days) for the self-registration system.

When an unauthorized user reaches the site, it now presents an invite redemption screen. During self-registration windows, this page leads with a big self registration button that gives the legacy screen.

Also, claude did about 85% of matching the original legacy site. That's very unnecessary friction for our vulnerable population, so I tweak the UI to use the same logo's, fonts and messaging as the legacy interface for both the check-in form and the away/dark page.

I also grab the site icon from the legacy site and use it in the app.

### UI changes

The organizer asked that the units sort numerically instead of alphabetically, so alphabetic labels will appear after the numeric ones.

I then add the ability to change your device name (in case it was skipped on registration and so you can identify which device 'this' one is.)

Now that I've been making requested changes instead of my own, I want to start adding workflow around it. The next request is to change the text on the check-in form (even though that is a change from the existing site).  So this time, I start with creating (the first) issue in the github repo, forking my code to it's own branch, making changes on the branch, issuing a pull request, testing the PR locally, merging the PR, deploying the code and deleting the work branch.

### Service hardening

I make some changes to the site configuration to block certain requests automatically, hiding the server information that is not needed and to push secure traffic by default.

## A bug!

The site domain name prefix is apparently only applied to the local browser and not to the site settings for everyone else. Another quick fix to choices claude did not ask. Now when the site admin sets the domain name, it will be reflected in the invite url lisks/qrcode.

### light mode

I notice that while I prefer dark mode, the check-in page is white, so maybe users want to choose. This involves adding the actual UI elements, which takes a few iterations to get ok. I ended up with a golden sun emoji to trigger light mode and half filled dark circle to get back to dark mode.

This hightlights a contrast ratio issue on the Manage Units page, so the deactivated units get a darker gray treatment and stand out from the blue much better.

### Migrations

This is a cuss word to some individuals. I look closer to how claude handled the four migrations that have happened so far and ..., well, it wasn't done in a sustainable way.

Getting a handle on that was easier than first proposed. There was only one, non-live installation so far, so I made 'this' version, version 0 and moved to a sequentially numbered migration system moving forward.

I also use a secret/token system in the deploy script to protect migration operations. The tradeoff is that for the moments between the code start deploying, until finished and the migration function being triggered by a url request, some users could see temporary errors.
This also replaces the previous system which checked for migrations on every request.

### Thundering herd

So it looks like claude naively built each parcel as it's own object with it's own update mechanism. That means on this large map, that every refresh is hundreds of requests to the server.  This is actually managable with so few users, but it's wasteful.

Streamline the backend requests to three: last check-in, away-windows, notes of all units instead of three requests per unit.

## Security review

So, I initially proposed this application on ChatGPT, with a discussion that sounded great, but when I went to implement things, ChatGPT didn't have the same tooling I have with Claude. So I instead had ChatGPT write a handoff document to give to claude to get me started.

Now it's time to check-up on how this session is working. I've tried to retain visibility into what I'm crafting, even though I'm in auto mode, by reviewing some of the code, and all of the feedback for assumptions to check or correct.

Let's make ChatGPT a first class citizen on my laptop (shiver) by installing it's actual client. Then I grant access to this one private repo and the local install and ask it to evaluate my progress and any issues that need to be solved.  I actually got a B+ with only one unexpected security issue (a race condition that would allow more than one redemption of an invite if they are all submitted at the same time in parallel) and two penalties around the legacy self-registration system.

### More UI updates

There are actually a number of tenants who are no-tech for their checkin, instead they put a sign with the day of the week in their window and a helper checks their window every day after the cutoff time (10am).  This requires a modification to the colors so that:

* Before the cutoff time, app unit's are yellow, sign units are orange.
* After the cutoff time, non-checked in app units are red, non-verified sign units are orange, and all checked-in/verified units are green.
* Dark/Away units are gray
* Deactivated units do not appear on the live screen

But I had a hard time telling orange from yellow and red, so I made a command decision to change to a purple, and then built a color selector UI for the settings panel so the site admin (in this case the organizer) can tweak them however they like, with an easy to apply color-blind safe theme.

This also required making a change to the staff screen, giving a toggle for app/sign mode in the units detail page.

## Gotcha

Oops, adding the color settings triggered a new migration, but now that migrations run from the depoly script intead of every request, nothing applies migrations to my development environment.  A very quick solution is a new local-migrations deploy script that I can run whenever the migrations change, mimicing the real process for live servers.

### Zoom issues

I noticed that when zoomed into the map, selecting a unit is not propery positioning the screen to the detail page.  This becomes a huge rabbit hole, fighting with multiple browsers in different sizes (phone/tablet/laptop) before I settle on drawing the details panel in the current viewport and skipping moving the page at all. This makes the panel render strangely based on how zoomed in the map is but it's the only real solution I can come up with.

Another issue lost the locate on map feature when open detail page became the search action. So split the button into two, a locate button and a check button.

Deactivated units that had an authorization for the unit, still showed the UI and allowed a unit to check-in (even though it was not visible anywhere). This fix now shows a message about being deactivated when trying to check in.

### Install button!

Since people are using devices, I added an install app/homescreen option to the app. It can be dismissed.

### Deploy script madness

While multitasking, I was paying more attention to the deploy process than I had and noticed that the script was deleting and uploading every file, every deploy, even if it had not changed. Worse, it was temporarily deleting a security definition file that for a split second, exposes the site to bad behavior.

That was a trivial fix that speeds up the deploy time as well.

### Forget me

I've deliberately reduced the attack surface to this app, and the only PII stored is the user name (which most users gets automatically set to "Unit ##").

Add a search/filter/delete interface for invitations and user devices and an account erasure tool that replaces the name with DELETED.

### Device maintenance

I added the abiity for a user to see and manage their devices, including to allow a device to see which one 'this' one is and to change it's name or revoke a device they no longer use.

And, if a user revokes a device they are using, transition immediately to the invite redemption screen.

### Site Recovery

Something I built in the begining is a Site Recovery Code, a long, hashed string that the site owner has to agree to have copied before site administration begins.  Without an email/sms registration path (more PII I'm trying to avoid, but will probably opt in later), this is the only in-band recovery method. Losing the site recovery key means you need file level access to the server.

Now I want to expose that interface, which will generate site owner inviites, rate limited to 5 attempts per 20 minutes.

Also, rather than adding another field/endpoint, I update the redemption screen to accept a 'gate code', typein in 'RECOVER-ME' as the redemption code reveals the site recovery entry form

### Reduce PII

There is a trade-off between useful historic data and creepily leaving the data to be mined if it's compromised.  I create a daily purge mechanism that removes check-in data and dark/away periods that have past a retention period (default 90 days, 0 for forever).

Notes and devices stay because the initial location includes people who may travel for more than one season.

### Hide Device UI from tenants

Well, feedback is that the devices tab will only confuse check-in only tenants and ask to have it removed and move device management to the staff/helper/owner roles.

And, since this leaves check-in as the only page option, remove the page selector from the top completely on a Unit only user.  This may mean to add helper/staff role to a Unit user, you have to revoke their Unit device, Generate a helper/staff invite and then add their Unit so they get the simple unit checkin screen in addition to the live screens.

## Target to go live

The day has been chosen to enroll and train the helpers. The training went well and helpers are excited for changes. Sunday night at 10pm, the site will go live so it's ready for midnight check-ins.  I do some of the prep in advance, confirming that the site will continue to run properly when moving from /v2/ to the main domain without deploying two copies of the code.

### Missing notes for helpers

So it looks like a helper only role can not see notes. This then progressed to not being able to create notes and then not being able to delete other users notes.

### Updated app icon

There was a new app logo dropped by the organizer. This means updating the assets in the app, the manifest, generating the correct file formats from the .ico icon file.  I then go ahead and push that as the whole site default root site now since they cache for a long time.

This involves a hiccup. I need a way to re-trigger the 'install an app' button, but I really don't want to add yet another button. My current solution is another invite easter egg. "INSTALL-NOW" will attempt to re-activate the button, which I have various success with on different browsers.  It's a start, the park is targetting chrome and safari, the two I have mostly been using (along with Brave and Firefox). The organizer also tests with Opera.

### Unit number

I update the check-in screen to show the unit number, so a helper can instantly tell if the user self-registered incorrectly. It does take a staff role to revoke the incorrect unit number and generate a new invite code.

I also determine during testing, that a 'helper only' user can see the devices tab so they can add their own devices and to enable invite upgrades, but the label for that page says it generates invites.  Small but important correction.

### Invite/User list size

Now that there are so many more users and invites, the page gets really long. It takes a couple of tries to get this one to work correctly, which adds a scroll bar to each height bound list.

This revealed another very clicky requirement for cleaning out different stages of invites, which gets fixed by a 'Delete all X' that is anabled only when 'Show All' is not selected.  Delete All for pending also skips the revoke step that is currently required for individual removals.

### Refresh icon for the live map

The current deployment has one volunteer active each day, so there isn't really a need to solve for multiple helpers to sync data updates (a feature we can't add to this hosting solution). It is possible that residents may still be updating during the period after the cutoff time.  Rather than forcing a page refresh, I add a refresh button that will just update the checkin/notes/away data.

While this host doesn't allow nodejs services, I want to add that feature in the future and I plumb the setting to enable it for testing when I get my hosting server up.  This service will run as a sidecar with the php backend, only providing a simple, no-auth broadcast signal received from the backend when something has updated, triggering a regular authenticated update.

## Going live!

I have an execution plan.  It begins by backing everything up, especially because there are breaking changes happening (like moving the database out of public) and removing the old site.  Then I deploy the site to the root path.

As expected, this evolved past the plan.  First, the organizer requests that the copyright and terms of service are restored. This terms just ends up being a mobirise default document so something else to add to the list of todos.

Then we try to make the self-registration form the default login for the day but... using .htaccess for this redirect breaks the svelte site and only loads the form.  That means we need to move the self-regisration inside the app instead of a separate link.

Claude of course doesn't remember how important the styling choices were for this page and completely botches it. Instead of tweaking again, I make claude backup and start over.

### last device standing

During testing, the manager accidentally revoked all their devices and logged themselves out.  I've had similar mishaps before I labelled the devices and added this 'this' device highlighting, but, well, it's a problem that it's so easy.  I started to work on this earlier, to prevent the site-admin from removing their own account.  So this extends that thinking further so that, a user can't revoke access to the same device they are using (which could potentially log them out completely). Instead, they need to use another logged in device to revoke the access.

Of course, I anticipate testing where we want to actually cause this to happen, so I add yet another invite easter egg 'LASTMAN-OUT' that will revoke 'this' device after confirming.

Then I move the invite a tenant button up above invite a helper/staff/site since those are used less frequently.

### clean-up on isle 1

There are post-deploy tasks that need to be completed.  First up, change the deploy.sh script so that public_html is the default install directory and use a DEPLOY_PATH environment variable to set something like /v2/ moving forward.

Then it's time for the first round of file clean-up. Starting with the old mobirise site which had a few conflicting assets, so were moved to a subdirectory temporarily in case we had to revert.  Things went well today, with only 1 resident not checking in on time and being absent earlier in the day. So I turned off self-registration and removed the leftover files from the v2 install.

### hardening

At startup, the site displays the recovery key and a field for the site name. This is a race condition to be the first (and only) visitor to the site.  Alter the deploy script to generate a random token that is passed to the admin, who enters the code/follows the link to begin the site setup process.  This supports an ENV variable with a fallback to a local file for deployments that don't allow ENV.

Not allowing ENV also means, I need to do a similar setting to mark a deployed site as Production so that X-Test-Mode headers do not function to bypass live sites.

Finally, I remove the ability for staff to revoke a site-admin role/device, since this could lock the site out.

### Generalizing

Organically growing from an original site leaves the codebase littered with the original park information and map.

I start by generating a demo map. This ends up being a script that generates an svg map, with 200+ units, converts the image to png and generates a matching importable map json that can be synced to the backend.

Then I added a new setting that allows a image file to be uploaded. To prevent scrapeing of a known map url, I generate a random asset name so we can serve via static url, but the settings panel remembers the original uploaded filename.  Deploying this change to production allows me to upload the current map (overridding the default map) and THEN I can replace the default map with the demo map.

I mostly needed this map so I can have non-real data to use for screenshots and demos.

Replacing the map means that we have to reset the park. This will reset most everthing because of foreign database keys, so I add a new owner only invite easter egg "ARMAGEDDON?" to trigger the confirm dialog.

### Recording

I'm going to be recording screen sessions of me going through many of the steps in the site administration and regular usage, and I anticipate it requiring more than one take to get it right.  So I introduce an owner easter egg 'DRIVEDAISY!', that enables a new UI page that has buttons for each step in the script (currently 5 steps). For safety, this new setting is only allowed if the default map is being used or if the uploaded map file name is called 'enable_demo.png'.

These steps do large scale changes to automate screen updates, activating a random set of units, some using signs and app, changing the cutoff time so units flip status, and marking units as checked in so the statuses clear.

Two of the steps manipulate the cutoff time. The first sets the cutoff time to now + 1 hour so they show as pending regardless of the time of recording. The second sets the cutoff time to now - 15 minutes so non checked in units show as expired. Since this touches on time, I take the moment to add cutoff time and timezone fields to the Settings UI.

### Moar clean-up

There was a temporary site transition piece built that was only used one day as a backup in case things went awry.  Remove that code, with it's hard-coded public secrets, a new post-event hook (for offsite auditing) might happen some day but no need to keep this code around.

The deploy script is currently bound to the live site. Refactoring a bit and adding environment variable overrides (so linux/docker works) for the four main security codes gives me the flexbility to add a Demo copy of the site on another domain name so we can test without affecting the live site.

When a user loses access on a device, a new invite code was created which ends up creating a duplicate account.  To make it visible which device is the 'current' device, display the last usage time for each registered device and user. then duplicates/expired devices and roles can be removed so they don't show up on the schedule. Then allow staff/owner to generate a new device code for an existing users account. Staff can not add a device code for an owner.

### Shift coordination

There are currently two daily activities that someone is reponsible for. Coordinator and Rounds. The coordinator reviews the map and follows up on discrepencies. The Rounds person(s) visit each of the window-sign people and updates the notes/status of each purple house.

I created a monthly view that shows each of the roles by day and open or covered (by who). Clicking on a day gives you an interface to claim the day (with options for a repeating/time limited range). Staff/owner can assign others to a role.

Things get tricky just after I released this feature, where I was requested to change the name of the 'Driver' role to 'Rounds' role. Migrations make this more stupid than necessary for claude, but I redirected this to a reverted change and a temporary script to update the dev/demo/prod services. I don't want to have a huge block of tests around a fragile change that has only a little actual data so far and won't affect any future site. Ripping that out feels good.

There was a few iterations of getting the process of claiming an already claimed shift with a repeating schedule to work. A tiny note, it was requested that the repeating option defaults to no end-date.

Then I add Step #6 to the demo control panel that populates a few helpers and assigns them MOST of the shifts for the next 12 weeks so that the alert shows how many shifts need to be filled.

Finally, highlight unclaimed shifts in the next 12 weeks on the calendar, and because I keep getting questions about it, add a notice on the page that says the auto-scheduler only forecasts 12 weeks in advance.

### Check-in only Easter Egg

For testing or edge-cases, it would be nice to have a way for a unit-only user to access the People and Devices tab (to manage their own devices or add a new role). On the check-in screen, tap the house icon four times and then click the Dark/Away schedule link at the bottom of the page to reveal the People and Devices screen.

### Note highlight colors

It turns out that we ran out of colors again. Note highlights are vermillion, which is the same color as expired units, making it difficult to tell that the border matches (the border isn't black).  I make this blue (unused on the live map) and gave it a color setting in the control panel.

This, of course conflicts with the blue activated units on the Manage Units screen, so for now I just switch the note highlight to vermillion here and not need another settings yet.

### Application icons

Two things came up with the site icons. 1) Anyone who installed the preview version of the site got the prototype icon, so cache-bust that. and 2) there is a desire to install both the demo and the live app and tell them apart, so I build an override capability into the deploy script and generate a temporary icon with a Demo banner across it.  When the organizer does finally generate icons, I have him create me a Dev icon as well.

Apple makes things complicated, the regular browser and a PWA installed 'app' do not share data profiles. So I explicitly warn a new user to install the app first, and THEN use the invite code to log in, otherwise the OS will eat the first invitation and another will have to be generated for the PWA installed app.

And since I'm here, I generate a link-preview card for web services (facebook/discord/etc) about the site.

### Zoom controls

Feedback is that there isn't enough zoom available to read the map. This means we're exceeding what's built into the browser, sad face.

A few things here.  Add a zoom control to the top left of the screen, not ON the map where a unit under the buttons would be hard to click. Also, allow the map to scroll past the edge a little bit in case there are units at the edge of the map.

After applying this method to the Live map, I apply the consistent change to both the Manage Units screen and the Map Editor.  

Touching the Map Editor inspires me to work a little on the Tablet view of the editor, which does not currently work (screen placement issue).  It takes a few iterations, but I get a landscape tablet view to support all the functions needed. Although this makes it POSSIBLE to edit the map without a desktop, it is not a great experience without a keyboard.

### E-Tag and api requests

Total overoptimization about to happen, it probably makes no sense on a site this small, however, it bugs me that the API requests scan much of the database on each request.  I already built a data-retention process to purge old data, so that potentially limits the load.  I happen to already store 'last updated' and 'last deleted' flags for each of the data components that make up the API request.  That means I can synthesize an E-Tag for the current state and return 304 NotModified when there hasn't been an update, saving queries and bandwidth (again, miniscule for one site).

This also means I don't have to wire up lots of endpoints to fire and track a change notification service I had planed for live updates. Living with a 30 second delay seems like a worthwhile solution, especially with the refresh button can cache-bust around a theoretical stuck tag.

### Tiny schedule changes

While the schedule is looking mostly complete, I add a highlight to show unclaimed shifts (within the 12 weeks auto-schedule), and I add a highlight for 'TODAY' on the calendar.

### Authentication persistence

After a few weeks of tests, apple users have to keep getting authentication codes. This ends up being more limitations apple imposes on apps. It takes a few iterations to get to a point where testing can try again, however, existing users will probably have to delete, reinstall and reauthenticate their devices 'one more time'.

### Map legend, Docker and live updates

Added a requested legend for the map screens.

I've used docker as a demo/testing environment, but I haven't actually built a deployable container that I can launch from a container repo.  I want to handle docker image builds on my mac (even though multiarchtecture builds take 5 minutes each) and deploy to github's container repo so I can just launch containers on my own server.  This get's me an actual stand-alone instance.

And then, I found a better solution to the notification service (using a prebuilt container called Mercure), by hosting the notification service on an external server (since boomhost doesn't support docker) so that production and demo now get live map updates. 

I also added a subtle version number to the footer so I can tell when a cache is busted.

## PII

Well, it's come time to add more data to the app. This adds Name, phone number, emergency contact, pet names, and key location information to each unit. (This is in progress towards automation of text/voice calls for checkins without smart devices).

I have an inconsistent CSV that is the emergency backup copy of the site data in case the app stops functioning or the internet is out. I work on an import that gets much of the data, however, there were issues getting some (iOS) users enrolled and they ended up with multiple accounts. This caused more problems with importing, but I'm leaving the resolution to this issue to the users.  I did create a new export feature so the emergency backup copy can be kept up to date.

So the user names, phone numbers, pet names and key location are encrypted in the database. This kinda sucks because environment variables are not properly supported on Boomhost, so they encryption keys need to be kept in files on the server (no worse exposure than a .env file) in that case.  That requires a hack to copy the database and the encryption keys if on a site without environment variables.

Staff and helpers can view and edits most fields, they key location field is only visible to the site owner, other views note if that field is set.  Tenants can modify their own account information, key information isn't visible in case a helper is looking at the screen.

### Twilio

So, to begin actual integration, I choose the Twilio (and compatible). This will give me text and voice capabilities. However, ... this begins a chain of events that iterates on me needing a marketing site https://chap.weave.sh so that I can have my phone number reviewed for my campaign.

Mocking this site causes me to redo the fake map, the original generated version screams of slop AI, so I tweak it a bit until it looks less dumb.


