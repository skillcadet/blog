---
layout: post
title: "Building CHAP: A Community Check-In System That Keeps Evolving"
date: 2026-11-23
---

**Project:** [CHAP — Community Home Alone Program](https://chap.weave.sh)  
**Status:** Active development. Voice and SMS features are **not operational**; phone-number authorization is pending.

My mother lives in a community with a Home Alone program. Participating residents check in each day, and volunteers follow up when someone hasn't checked in. Some residents use a web form; others put a day-of-the-week sign in a window for a volunteer to verify in person.

She initially asked whether I could help provide backup for the person maintaining the existing system. That sounded like a straightforward continuity problem. Once I saw the workflow, it became clear that there was an opportunity to make the volunteers' daily work easier without asking residents to learn an entirely new process.

This is a development journal turned into a case study, not a claim that the software is finished. CHAP is in active use and active development, and I'm still working through its limitations.

## Start with the people, not the stack

The original process involved checking submitted forms and email, comparing them against participating units, and following up on missing check-ins. The organizer and a small group of volunteers carried that work.

My first contribution was deliberately unglamorous: help the organizer put the existing site under version control, establish a backup, and make it possible for someone else to maintain it. That reduced the risk of a single person being the only one who could keep the program running.

Then I proposed a map-based view. Instead of reconciling records by hand, a volunteer could look at the participating units and see which ones needed attention.

That sounds simple, but it shaped nearly every technical decision afterward.

## Build around the constraints

The original site lived on inexpensive shared hosting, where PHP was practical but a persistent Node.js application server wasn't. I chose a Svelte 5 frontend with a PHP API and SQLite storage. Vite supports local development; it isn't part of the production serving stack.

I used Docker locally so I could iterate without experimenting on the community's live site.

The map was central to the experience, so I started by building an editor that could turn an existing site-map image into individually selectable parcels. Automatic boundary detection helped, but it wasn't magic. I still had to correct shapes, label hundreds of units, and make the editing workflow tolerable with keyboard shortcuts and better focus behavior.

The result wasn't just a map graphic. It was a way to connect the physical layout volunteers already understood to the daily check-in process.

## Make the easy path genuinely easy

A resident using the original web form had to select a unit and the day. With a device associated with a unit, the new check-in can be reduced to opening the page and confirming they're okay.

But not everyone uses a phone or computer to check in. The window-sign process remained important, so CHAP had to represent both kinds of participation rather than treating the less technical residents as exceptions.

Volunteer feedback led to many small but consequential changes: clearer status colors, easier unit lookup, better map zooming, simpler check-in screens, and adjustments for mobile browsers and installed web apps.

I also added role-specific interfaces. Residents should see a straightforward check-in experience; helpers need a live operational view; staff need to manage participation and invitations; and site administrators need configuration and recovery tools.

## What AI-assisted development got right — and wrong

I built much of CHAP while pairing with Claude, using AI-generated code as a starting point rather than assuming it was correct.

That accelerated the first versions of the map editor and administrative interfaces. It also produced mistakes I had to catch: overlapping controls, assumptions about deployment paths, inefficient per-parcel requests, brittle database migrations, and UI changes that drifted away from what residents already knew.

One particularly instructive problem was a map refresh strategy that made requests for individual parcels. It worked at small scale but created unnecessary traffic. I changed the design to retrieve related status data in batches, and later experimented with conditional requests to avoid transferring unchanged information.

The lesson hasn't been that AI can replace engineering judgment. It's that faster implementation makes review, testing, and understanding the surrounding system more important.

## Reliability, privacy, and operational tradeoffs

A community check-in system has to be understandable and maintainable by people other than its developer.

That meant thinking about administrator recovery, device invitations and revocation, role boundaries, audit trails, backups, and database migrations. It also meant learning from mistakes in deployment scripts and making sure a routine update wouldn't unnecessarily expose files or disrupt the live service.

As the project expanded, it began to handle more sensitive resident and emergency-contact information. I introduced encrypted storage for selected fields and access restrictions by role. Those are risk-reduction measures, not a declaration that the application has completed an independent security or compliance review.

I also built tools for demonstration and testing with synthetic map data, so I can show the interface without publishing residents' information.

## From daily check-ins to volunteer coordination

The system grew beyond the original map. Volunteers coordinate two different kinds of daily work: reviewing check-in exceptions and making rounds to verify window signs. CHAP now includes scheduling features to help cover those shifts.

The underlying design still has to balance useful automation with the reality that a person must make the follow-up decisions.

## What's working, and what's next

CHAP has progressed from a backup plan for an existing site into a deployed, evolving application with map-based check-in visibility, resident and staff workflows, and volunteer scheduling. I'm continuing to refine its usability, reliability, and deployment options.

A major area of work is supporting people who cannot or do not want to use the web interface. I'm integrating voice and SMS capabilities, but **those features are not working yet**. Phone-number authorization is pending, and they should not be treated as available check-in channels.

The project has also pushed me to revisit questions about data retention, access to sensitive information, cross-device authentication, and how to test changes safely while a community depends on the service.

You can find the project's public-facing site at [chap.weave.sh](https://chap.weave.sh). I'll continue documenting the engineering decisions and lessons as development progresses.

## What I've learned so far

The most useful improvements haven't always been the most technically ambitious ones. A clearer status indicator, a familiar check-in screen, or a reliable handoff between volunteers can matter more than a clever backend feature.

The technical challenge is real, but the measure of success is whether the people running the program can spend less time reconciling information and more time responding when someone may need help.
