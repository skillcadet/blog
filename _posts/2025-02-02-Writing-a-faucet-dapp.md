---
layout: post
title: Writing a Faucet Dapp for Autonomi Testing
---

## What?

The following glossary attempts to make this clearer:

* [Autonomi](https://www.autonomi.com) - Provides a decentralized storage network that allows you to pay once to permanently store and then retrieve information free of charge.
* Faucet Dapp - Is a term used for a system that rewards users with free cryptocurrencies.


In our use case, "Testing Autonomi" means to upload data to the network, which requires two separate cryptocurrencies:
* ANT - Autonomi Network Token is the currency used to pay and reward nodes for uploading data.
* ETH - Ethereum is another currency we must pay to use ANT (called gas).


The code produced by this adventure can be found at [this repo](https://github.com/iweave/antfaucet)

## Autonomi Pre-launch

As of this writing, the Real token has not launched (target is soon, Feb ~~6~~11, 2025), so the only way to gain ANT has been to [run nodes](https://docs.autonomi.com/getting-started) on the test networks, earning little bits along the way.  Since the nodes are allocated randomly, it can be challenging to earn enough tokens to practically upload a file.  For example, it takes roughly 12 rewards to earn enough ANT for the smallest file to upload, which can take more than a week to earn at the current rate.


To allow users to more quickly and easily test uploads, we needed a simple way to grant test tokens that have already been earned for others to use. After a discord member suggested that there be a faucet, I thought, "I can learn something even if I don't succeed. I'll give it a try!"

## The setup

First up, I needed to find out how to transfer tokens in Python. ANT is something called ERC-20 that lives on a second-layer network called Arbitrum (which is compatible with Ethereum, a popular cryptocurrency.) I was confident that transferring ETH would be well documented, so I looked for Python examples that transferred ERC-20 tokens. After more searching and testing of code than I hoped, I finally stumbled on [this](https://web3-ethereum-defi.readthedocs.io/tutorials/transfer.html) page that just worked. A small bit of research into the Web3 API that the example was from showed the slight change needed to also send ETH from the command line.


That gets us most of the way there, the rest is wrapping it all into something accessable to others.

## Getting started

I've used Python for simple task automation recently over using other previous tools like bash/perl/php as it provides a lot of power and is often installed on systems by default. I've also done enough with Python to know to use [virtual environments](https://docs.python.org/3/library/venv.html) to localize package installation. So I start with creating and activating a Python 3 environment to paste my example.

### Topping up

I know about faucets, in part, because I have been using [one](https://www.alchemy.com/faucets/arbitrum-sepolia) to fund my own testing on the web3 stuff. Since their faucet has been reliable, I decided to use [Alchemy's](https://www.alchemy.com) API endpoints to power my transactions.  This faucet will definitely fit within Alchemy's free tier. To have some funds to play with, I topped up my wallet with Alchemy's faucet.  I had already been given some test ANT and ETH by the person who suggested the faucet, but I wanted to save that for after the faucet was working.

### Linking to Alchemy

I logged into Alchemy and in my dashboard, I created an App endpoint connected to the Arbitrum Sepolia network that I used to fill in the endpoint from the example. To get the sample code to work, I also had to import the private key of my wallet. Since this code was running on my laptop, I wasn't worried about the key leaking. 


Finally, I used the ERC_20 address of the ANT token for the contract information in the sample code.

### Flask

Now, outside of tutorials for other things, I've not done much front-end anything, mostly I have worked on the back-end of things. I knew I wanted to turn to Flask as a simple front-end to handle URLs, and an early hit found me [an example](https://www.askpython.com/python-modules/flask/flask-forms) to use to get started.  I created my template html files and wired up the simple main.py to launch my form. All I needed was a text box for the wallet address and a submit button.


Once I could see my form data rendered, I could determine how to interact with the value as provided by Flask.

### Building out the main function

Now to get to the meat of it. I created a function called drip_coins() that returns a payload with two items, "status" and "reason". First I start with a fixed status of False and a 'testing' result. My goal is to pass the results to the render form. If the status is not "True" (everything else), render the 'fail.html' template which shows all the values returned.  Once I have that wired up I begin to work the code from the transaction sample into drip_coins().


First up, there are `asserts` littered in the code. Anything that occurs after the app is launched would cause the app to crash, so move some of the sample code (like the part that loads environment variables) out of the function and into the main body of the code.  That still leaves us with things that will be defined after the code launches, like the user's wallet address.


We'll add other validations before we get here, but one of the first things the code does is check the wallet address for a checksum (the way the upper and lowercase characters are defined in an address help to validate that the address is correct).  If we fail the check here, return a failed response. I like to test edge cases along the way, so I try various good and bad wallet addresses to verify my check is good.


Next, we include the code up to the point that checks the token and ETH balances. This seems like a very reasonable checkpoint, let's return a result showing the amount of token and ETH using the False status to display the values.


The sample code reported the chain ID of the blockchain, we can use that as another validation that things are correctly configured. Add a header defining the chain ID and return a False status if the values don't match.


Instead of the user entering the amount to transfer, we can wire that to a variable we set in the header. In case we have to iterate a lot, use a small value (I used 0.00000001 to start and incremented the value between runs so I could tell the transactions apart on the blockchain reporter).


Now, we're finally doing transactions. Return the ETH and ANT transaction IDs as False results.


That looks good. Let's create a "success.html" to give a render target for our good result!  Then, of course, we need to change the status of the good results to True so we can render our success page. :)

### Validating data

It's getting serious now, we have a function to protect. I created a validate_request() function and passed it the form_data. We know a wallet address can be between 0 and 40 characters long and only contain hexadecimal characters (upper and lowercase) and the letter x/X (to notate it is a hex number), so that's our first check.

### Persistence

A common feature of a faucet is either a one time use (per address) or a cooling-off period/activity required before the next drip. That requires some form of data persistence. Luckily, for our simple needs, [sqlite3](https://sqlite.org/) is a robust tool that is included with Python.


After we define the database in the header, we need a way to know that the database is ready to go when we launch the server.  So I created a prepare_faucet_db() function that tries to get the first row from the database. If that fails, we immediately try to set up the database for the first time by creating the structure and inserting a test record. I then invoke this function when the app first launches so we crash right away if we can't get/create the database.


The next step is to check if the wallet already exists in the database (we know it doesn't), so I created the check_db() function to select any records in the database that match the wallet ID. I start with automatically returning True so that I can test the failure that the wallet already exists and set an appropriate message in my reason. Then I add the real check, which returns a False if there is no match.


And finally, I created the add_db() function to add transactions to the database. This causes subsequent transactions for the same wallet to be declined.

## Bringing it online

### hcaptcha

Now it's all fine and good to develop on a local computer, eventually we want others to test our creation, but I am running a little server that I don't want a thundering herd by someone writing a script to drain the faucet's wallet.  To protect the 'expensive' database calls and other complex processing, I want to protect the input form. Some search for a privacy-compliant option found me [hcaptcha](https://www.hcaptcha.com/), which required very little to set up. I had to register for the free service, create an endpoint, and add some JavaScript and a `<div>` tag to my web form template file. Hcaptcha also requires to be served from an HTTPS web page, so I had to set up that (described briefly below).


Following my normal steps, I created a check_hcapthca() function, set up a return False and a reason of the captcha failed, added the new function to validate_request.
This showed me the new variable that was returned by the form.  I added a check to make sure that the captcha was provided or we fail right away.


If the captcha is provided, we need to verify by making an API request to hcaptcha and getting confirmation that it's both valid and timely. This payload is quite large, ~~so I don't have any filtering on it and it's a potential vector for buffer overflows and wasting resources. Something for the future if this lasts more than a week/proof-of-concept.~~ Again, I start with the failure case hard coded, then add the API call, and if we get a success finally add the True status.

### HTTPS/SSL Let's Encrypt

I prefer [NGINX](https://nginx.org) as a web gateway which has support for [Let's Encrypt](https://letsencrypt.org) using [CertBot](https://certbot.eff.org/).


I log into a server with a public IP address (and a private address on my internal network), create a nginx configuration for http://ant.xd7.org, test my nginx configuration and reload nginx. Then I add the public address to DNS and wait a few minutes for the update to be published before starting Certbot to create an HTTPS certificate for my domain.


Now, I can change the upgraded configuration file to point to my Flask backend, check and reload nginx and I can access the faucet at [https://ant.xd7.org](https://ant.xd7.org)!

### Faucet Wallet

To get things going for real, I created a NEW wallet and transferred some initial funds from what had already been provided and then linked the wallet to the server by copying the private key over and ran a test transaction to my own wallet.

## Testing

I announced in the Autonomi discord channel that the faucet was up for testing and a few people went ahead and submitted their wallet addresses successfully. No one reported any issues. Then the user who asked for the faucet tested and gave the go-ahead to increase the faucet to .25ANT and .001ETH, so I reset the database, and deployed the changes.

## Gunicorn

I know that the built-in web server for Flask is not production ready (besides, it warns you every time you start the server), so I logged into the server, activated my Python virtual environment, installed gunicorn, and figured out how to launch my Flask app with gunicorn (which entailed refactoring a little to not pass the faucetdb as a parameter to the database calls as gunicorn was not creating the database object when importing the app).

## We're live!

But are we finished? Not one to leave a good thing alone, I wanted to check the faucets from my phone. But I don't want to impact the live instance with my testing, so I create a development server on a different port and set up another NGINX reverse proxy to activate the new internal node.

## Success/Drips

The success endpoint was pretty simple, it just counts the number of records (minus the initial test record) and displays it onscreen. The only difficult part was I had already used the 'success.html' template filename, so I renamed the endpoint from /success to /drips.  The next day, I used CSS to increase the font size to very large, so it renders visibly on a smartphone.

## Rate limit

After a little discussion, the lead user suggested a good idea for the endpoint... a Rate limit. I decided to start with 6 drips/hour as a reasonable velocity.


First, check_db() became too generic so was renamed to check_db_for_wallet(), and then a new function check_db_for_rate() was defined and set to automatically return a True. Then I added a simple query that checks the COUNT() of the number of transactions in the last RATE_WINDOW seconds is more than RATE_LIMIT in which case it returns the count as a True and False only if we are below the rate limit.

## Restore test transactions

It bugged me that I reset the database and lost the initial 'live' transactions. I initially chose that route so the testers could re-request a full faucet drip, but deleting the database was not the only option. The solution was something I already planned on, but didn't execute initially to stay focused on getting things testable.


So, I pulled up a copy of the database I had cloned early in my testing and created a Python list of transactions to re-insert.  Now I have a record of all the transactions that the faucet has succeeded with. But I didn't want the first users to be left out, which required one small change. In the query for existing transactions, I added TIME_HORIZON which is a unix timestamp of the last transaction from the test db, so only returns transactions newer than that horizon.


This could later be extended to allow the same wallet to request more than once with its own velocity (like once/week).


### Limit upload content length

Flask provides for a simple way to block unwanted data, MAX_CONTENT_LENGTH, which will return a 413 HTTP Response when exceeded.

### Parameterize SQL

A few of the queries were spiked with non-preferred syntax. I cleaned those up by parameterizing the statements.

### Integrate MetaMask wallet

A feature I see in some dapps, is the ability to connect to a MetaMask wallet.  Digging around I discovered [this](https://github.com/RishabKattimani/MetaMaskWebApp) 3 year old example, which showed a very simple solution.

After getting the wallet address to auto-populate the form, I extended the code to turn the input field in to a drop down list if more than wallet address is connected.

### Favicon

I am a little annoyed by the failed requests for the favicon.ico file, so I created a 'static' directory, uploaded a copy of my logo and set the image in the form header.

### Now finished?

There isn't a need to view transactions since they are all visible on the blockchain.  Extending the faucet to have allow multiple drips doesn't fit in the current use, so MAYBE we're done... for now. :)

### Donate Button!

I added the wallet address to the page, with a button to copy the address to your clipboard.

### Adding transaction information

I accidentally committed some testing harness when bringing code from the dev environment, which created non-existent transactions while validating the live faucet. To make that transparent, I'm now adding the hash for both transactions.

If we detect that the test harness is active, record 0.0 for drip amounts since we didn't do an actual transaction.

In this step, I created a migration sql file and also updated the broken transactions to 0.0 eth/ant.