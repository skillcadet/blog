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

