---
title: "Credhub WebUI"
date: 2018-12-16T22:58:47+11:00
---

I've been playing around with BUCC (https://github.com/starkandwayne/bucc) which is a really cool and easy way to deploy a BOSH director with UAA, Concourse, and Credhub. And I started thinking about how hard it is for not-so-technical people to update credentials, or retrieve credentials from Credhub. Not everyone is good with the CLI tools, or can't use them due to workstations, but they can access a VPN and use a browser.
<!--more-->

## Enter, Credhub WebUI.
Or, atleast an idea about it. Quick searches on google and github lead me to nothing except one person raising an issue for it not existing..

I set out reading the API documentation for Credhub and UAA, and then I looked at some basic tutorials on web services built using Go, and executing HTTP requests to see what was involved.

## Build one
I wrote a quick HTTP server and got it to render some templates. Good start, but not quite there yet.

To do anything with Credhub, we first need to authenticate via UAA. This was the first time I had ever actually looked into how to do this with Go. After some tooing and froing between different API documentation, and trawling through git repositories looking for example code, I finally got that bit to work. And themed the sign-in page to make it look legit.

![Authentication](/img/credhub-sign_in.png)

Now that we have authentication, we can call the Credhub API endpoints. I ended up just using `templates/html` for this, as it seemed simple. And it was, for the most part. Some things returned via the API don't render properly, so some JS/jQuery was needed to solve those problems. Others required me to add functions to the templates so that I could process the expiration date from an SSL certificate to be made visible to the user.

These parts were easy, create a template for each type of credential, then create the structure for the code, rinse repeat.

Basic functionality of Credhub is as follows, and these were added into the UI. Most features are there, like specifying subjet names for certificates, adding or removing special characters from generated passwords, etc..

* List Credentials
* View Credential, unseal it and use it
* Set Credential, where you can create new, or update existing
* Generate Credential, generate a random credential
* Delete Credential

Some things may be missing, but that's the beauty of opensource, you can add them and submit pull requests. Links at the bottom of the page.

## What next?
Well, since Credhub goes really well in the BOSH ecosystem, might as well make a BOSH release for it.

While we are at it, use Concourse to set up the pipelines to manage the build and creation of releases too, storing the credentials used inside of a Credhub!

So satisfying to have built this, even if I am the only one that might use it.

## Screenshots
Here are a couple of screenshots
![List Credentials](/img/credhub-list_creds.png)
![View Credential](/img/credhub-view_generated.png)

## Where to get it?
* Here (https://github.com/shreddedbacon/credhub-webui)
* or Here (https://github.com/shreddedbacon/credhub-webui-boshrelease)

