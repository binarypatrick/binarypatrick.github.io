---
layout: post
title: "Creating a super Outlook SPAM Rule"
date: 2021-03-23 22:40:00 -0500
category: "General"
tags: ["outlook", "spam"]
---

After signing up for several virtual conferences in 2020, I noticed my work inbox was becoming inundated with junk email. Now this being my work email, I don't use it for any personal correspondence, and frankly never respond to anyone outside of my work's domain. Using Outlook online's email rules, I was able to eliminate all SPAM from my inbox in one easy move.

<!--more-->

## Notes

So, I'll start off saying here my situation may be unique. I only care about email from people inside my organization. If that's not the case for you, this definitely won't work for you. I'll also say I have a lot of other rules that run ahead of this rule to pick up any external email that I might care about. Like notifications from Azure or external monitoring notifications. Those are important and have their own rules governing how they're handled.

## Into the Clouds!

These rules work best when created in the online version of the Outlook client. If they are created on a specific, local client, they will only run when that client is running. When created in the _cloud_ they will always run. My organization uses [Office 365](https://outlook.office365.com/mail/inbox) so I headed there.

## Find your rules

For me, the easiest way to pull up email rules is to click on the settings icon (gear/cog) in the top right. From there I click `View all Outlook settings` near the bottom of the panel that appears. This will pull up another menu. With `Mail` selected, I click `rules` from the options on the left. If you have other rules set up already, you should see them here.

## SPAM be gone

From here click `Add new rule`. A create rule screen will appear and you can start setting up the rule. I named my rule "Not from {organization} is Junk". Under condition choose `Apply to all messages`. This may seem wrong at first, but it will make more sense in a second.

![Rules](/assets/img/outlook-spam-rules.png)

Add the action to `Move to` the `Junk Email` folder, and for an added nicety, Add another action to set `Mark as read` too.

Lastly, and most importantly, set the exception. The rules here can be tailored to meet your needs, but for me, `Sender address includes` and then my {@organization.com} did the trick.

## Finally

Now my inbox is much leaner. Occasionally I will check the junk folder just to see if I missed anything, but so far I haven't.
