---
layout: post
title: Building Autonomi Transaction Reporter in Svelte
---

## The dreaded UI...

I have reached a crossroads where I need to render an interface for an application.

I've been enjoying Python, but alas, I'm about to address something that will need styling and client side updates. That will require CSS at it's least, but Javascript for sure.

I acknowledge that my Javascript skills revolve around one project (a weave module diagnostics reporter) I built way back in 2013.  So, I'm looking at starting basically fresh, albiet with programming experience in other languages.

I looked at frameworks and settled on ...

### Svelte

[Svelte](https://svelte.dev/) and specifically [SvelteKit](https://sveltekit.io/) are a Javascript framework that compiles down to only necessary code to the client and reactive componenets (should be straight forward to piece things together).

My first thought, going through the tutorials is, I don't know enough Javascript... however, I found an article that goes into the amount of Javascript needed to use ReactJS (A competing framework developed by Meta), so hopefully that get me sorted.

Back from the ReactJS Javascript overview. I'm in over my head. But maybe I can accomplish my goals anyway. Only one way to know for sure.

Svelte 5 launched less than a year ago, so even though the Svelte environment has been around a while, most of what I find are for the previous (breaking change) versions.

I found a grid component in [RevoGrid](https://rv-grid.com/guide/svelte/) that is a Svelte 5 component, so let's see what we (the Internet and I) can weave together.

### The Grid

> "The Grid"
>
> &mdash; <cite>Kevin Flynn - Tron: Legacy</cite>

The core of this problem is dispalying and editing the grid of hundreds to thousands of transactions faced by the Autonomi node operator. One of the challenges of which, is to spread incoming transactions that have multiple originating transactions into additional records. 

A trickier problem to solve is: "What happens with the introduction of transactions that preceed an already submitted FIFO (or any) solution". I believe you would need to resettle the transaction catalog and re-file for the affected years. Need to find a tax accountant to be sure. For now, I plan on creating settlement records for each receive and partial spend events
 in a kind of double entry system for reconciliation.

But then I hit my next issue. Testing RevoGrid is giving me an error. I try different examples and all are giving me a TypeError.

I do locate a sample repo on github that works, so I apply the same package.json versions to my own projects and ... no dice.  Then, I remove the package-lock.json file from the working example and boom, TypeError.

I find a previous issue on RevoGrid's github, with someone experiencing a TypeError recently, however there is no comment on how it was fixed.  I submit a new issue and miss a few days because I didn't get notified there was a reply to my issue.  I was asked to submit a boilerplate repo that showed the error and then I had to wait a couple more days before the issue was reported as resolved.

I tried to do an update on my test repo using the web example, still broken, I pulled the changes to my clone of the revogrid example and that worked, so that's a good sign.

In the meantime, I have purchased a svelte5 tutorial and am going through this trying to learn how to build my app.

### ~~Im~~possible Futures

This project manifested as my submission for [Autonomi](https://autonomi.com)'s[Impossible-Futures](https://impossible-futures.com) Origin Series competition.

My goal is a tool that will help resolve the hundreds to thousands of ETH and ANT transactions that can occur from hosting a node or using the network.  It is an unfortunate situation where the ecosystem we use, whether known cryptocurrencies or a native token are, in most locations, taxable events.

My prototype scenario includes only FIFO (First-In/First-Out), which will be the only option in the US starting with 2026. HIFO (Highest-In/First-Out), a variant of Specific In/Out tracking, and any other options will have to be considered later.

On April 22, 2025, I was notified that I was included in the first cohort and will be invited to create my project page using a google form. Huzzah!

I've updated my project page with a placeholder image I generated with AI. In very short order, I need to mockup a more realistic page and try to make it look about the same way with photoshop. For now, I'm waiting to see the resulting page that is supposed to update regularly from the google sheet.

### Course update

Ok, back from the first module... I think I'll be able to figure out the Javascript, but where I'm feeling lost is CSS.  CSS is very flexible and I don't know the language of design in a way to apply it towards what I want (I don't know how to describe what I want in a consice way). I do appreciate how Svelte isolates CSS inside components, but I am going to have to take some styling lessons to make things look friendly.

### Javascript, The Hard Way
I started module two, which is all typescript (I was warned the last two modules would be). Ok. Let's regroup.

A recommended course I considered taking before was, Python The Hard Way, but I'm pivoting to Javascript for ATR.  Luckily, there is an Alpha version of the Javascript version of his course.

I've seen in the discord that this course isn't complete, so ugh, maybe I overpaid for it? $50 is more than I could be paying. This course was started in 2022, and appears stalled. It actually seems like it will/would use Svelte, but it'll be pre-svelte5. The fundamentals should be mostly current, and I'm confident this course will leave me with plenty to still learn even if it was complete. If the book is a bust, unlikely, I have 30 days for a refund. I want this course to work well enough that I can share it with others.

### ~~Im~~possible Futures Update

After 2 days with no votes, I created a forum post and a X tweet and woke up the next morning to being in first place. :)  This ranking lasted for 2 days, although the way the votes were cast likely disqualified me from winning. Luckily, I planned on finishing this anyway, I just needed the reward money to pay for the crypto tax consultant.  I've not given up hope, but there are a lot of good projects competing. I survived another day before being pushed out of the top 12 leaders.

Well, the day has arrived and while ATR took first place in the rankings, the way the rules are interpreted means that ATR was disqualified from proceeding to the next round.  At least I have no pressure to launch mid July. Unfortunately there is some unrest in how ATR vote transactions were refunded.

I have run out of content for Javascript the Hard Way, doh. Back to the Svelte 5 Typescript course. Onward and forward.

