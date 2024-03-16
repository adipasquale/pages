---
title: Leave Gmail in 10 steps
date: 2018-12-02
categories: en
tags: internet
summary: In exchange for free mails, would you let your postman open your letters, read them, and insert ads related to their contents?
---

- updated on 25/12/2018 : [*added link to Fastmail's post about AABill*](https://github.com/adipasquale/blog.dipasquale.fr/commit/93819a06f0551f01ec4304ba76c83705f38fbb36)


> In exchange for free mails, would you let your postman open your letters, read them, and insert ads related to their contents?

This is a common analogy for Gmail's model. The privacy implications of using Gmail actually go much farther than that, because Gmail is not only your postman (with Gmail), it also owns your car ( your web browser), your address book (with Search), your TV (with YouTube), it's a great land owner (with AdSense)… And there are many reasons not to trust Google, [especially with your privacy](http://precursorblog.com/?q=content/googles-top-35-privacy-scandals).

**Important edit**: As many commenters pointed out, this analogy is actually not correct anymore, as [Google stopped reading your emails for ad-personalization](https://techcentral.co.za/google-will-stop-reading-e-mail/75215/) in 2017, sorry I missed that news. However, they [do still read](https://theoutline.com/post/4524/remember-when-google-said-it-would-stop-reading-your-email) them in order to "customize search results, better detect spam and malware".

Migrating away from any email service, like changing addresses in real world life, is always going to be tedious. I did not find that Gmail makes it particularly harder, but I hope this guide can help you.

Disclaimer : This guide will not help you find alternatives for the Gmail-specific features, like labels, snoozing, bundles, suggested replies and so on.


## Step 1 : Get a new mail address and provider

First, I strongly suggest you buy a domain name, so that you really own your mail address, and don't depend on any service. You can purchase one at [Gandi](https://www.gandi.net/).

From there on, you could use Gandi's mail servers, but I recommend you use [Fastmail's](https://fastmail.com/). It's not free, but it will be much simpler to setup, faster, they provide push notifications for mobile, their spam filter is better, their web interface is better, and it's the most popular privacy-respecting service.

**Edit**: I am not affiliated to Fastmail in any way, and I do not want this article to look like an ad!
Also, as [HN commenters](https://news.ycombinator.com/item?id=18633216) and [@mediafinger](https://twitter.com/mediafinger/status/1071325185364672513) pointed out, Fastmail is an Australian company, where they just passed [a new bill](https://www.nytimes.com/2018/12/06/world/australia/encryption-bill-nauru.html), which is concerning for privacy.
Fastmail stated [in a blog post](https://fastmail.blog/2018/12/21/advocating-for-privacy-aabill-australia/)  that this law does not impact them though.
Have a look at other privacy-caring providers like [mailbox.org](https://mailbox.org/), [runbox.com](https://runbox.com/),[posteo.de](https://posteo.de/) and [startmail.com](https://www.startmail.com/), they may very well be cheaper too.

[Signing up with Fastmail](https://www.fastmail.com/signup/) is very simple. If you bought a domain name, you can use their [easy-setup option](https://www.fastmail.com/help/receive/domains-setup-nsmx.html) where you use their name servers and they do all the hardlifting. You will still be able to add custom DNS entries later if you want to, for your personal website or blog. If you did not buy a domain name, you can also create a mail address ending with one of theirs : sent.com, fastmail.com…

## Step 2 : Clients setup and update round

By client, here, I mean the applications that you use to read and write mail. Gmail has both a web client (a website) and mobile apps.

There are lots of options for mail clients. Fastmail comes with a web client too, that is quite different from Gmail and less featureful, but it works well.

{% image "./fastmail-web-interface.png", "Fastmail's web interface" %}

Windows 10+, OS X and iOS come with decent native mail clients, I suggest you start with these. Android on the other hand now has Gmail as the default mail app, so you will have to try a new one. Fastmail has some [great guides](https://www.fastmail.com/help/clients/applist.html) for setupping these mail clients and it's sometimes as easy as scanning a personalized QR-Code.

You can try sending a mail from your Gmail to your new mail and check that you are receiving it on all the devices that you care about. Once you are up and running with your new address, it's the perfect moment to try and update your mail address on the services you use daily (Social networks, Github, Amazon…). If you use a password manager it can help you identify the services you have forgot.

## Step 3 : Delete useless mails from Gmail

I strongly suggest that you take the time to clean up the clutter you have accumulated in your Gmail before anything. Here are some reasons to do so :

- migrating your mails will be faster
- you will not reach the storage limit on your new provider as fast
- it feels good :)

It’s important to do it before migrating or else you’ll be cleaning up 2 mailboxes (I did and lost a lot of time).

Look at your space usage for mail on [this Google page](https://drive.Google.com/settings/storage), by clicking on the details link. If it looks reasonable, you may skip this step.

The first way to find useless mails is easy : look for mails with big attachments. You can use this Gmail search query `size:5MB` or any other threshold.

The second way is trickier : you want to look for redundant, similar mails : facebook notifications, Google alerts, GitHub comments… This can represent a very high number of mail and thus of space. Surprisingly I could not find a good tool to identify these mails. It cannot be done with Google queries. There is a plug-in in Thunderbird but it was very buggy for me and opening Thunderbird feels like travelling back to the 90s 😢 The best way I found was to use python's mbox files support out of the box. I published a small script called [mbox-sender-frequency](https://github.com/adipasquale/mbox-sender-frequency). It lists all addresses that have sent over 100 mails to you. You can then filter and drop them on Gmail.

![example usage of mbox-sender-frequency](https://i.imgur.com/isCPq3N.png)

## Step 4 : Export your data from Google and archive it

I have to hand it out to Google they make it quite easy to export your mails. Head to [Google Takeout](https://takeout.Google.com) and select mails. You’ll need to have it locally so I suggest you take the download option. They’ll mail you a link once they have gathered it all in a nice mbox file.

I suggest you store this mbox archive somewhere safe, ie. not a hard drive :) I went with S3 Cloud Storage. You want to keep this file for a long while, maybe a year or so.

## Step 5 : Migrate all mails

Important : **do not yet use Fastmail's Identities & Fetch feature**. It would be slower and less reliable at this point.

Fastmail has a very nice Email import tool, that will fetch mail from Google's IMAP servers and import them to your new account. To use it, follow [these very clear instructions by Fastmail](https://www.fastmail.com/help/receive/migratemail.html). Do use the 'no duplicate' checkbox, because mails with labels would be fetched multiple times otherwise.

This step can take a while, a whole night for me. Once it's done, do give a look at the logs as you do not want to miss anything. You should also have a look at your mails, browsing to the oldest is a good idea.

## Step 6 : Delete all migrated mails from Gmail

Once you are confident you have everything on your new account and have backed up the archive file, you may go back to  and delete all your mails. That is a very thrilling step!

Note : Because the migration process takes a while, you may have received mail to your Gmail account between steps 5 and 6, and that mail was not migrated to your new account. In that case, you may want to limit the mails you are deleting using a search filter.

## Step 7 : Synchronize future Gmail mails to Fastmail

You can now setup the [Identities & Fetch](https://www.fastmail.com/help/receive/fetchotheremail.html) feature with your Gmail account. All the mails received during the migration, and the future incoming ones to Gmail will now be synchronized to your new account. You can also setup a sender alias to write mails with your old Gmail address, if necessary.

You are ready to remove the Gmail apps from all your devices, you are almost free!

## Step 8 : Monitor mails still sent to your Gmail account

The idea is to reduce more and more the mails you receive to your Gmail address. To do so, you can search `to:youroldaddress@.com` on Fastmail web interface, and save it. Then, go check out the results regularly and try and make sure that you will not receive similar mails in the future.

If it’s an automated mail from a service you use, go update your address (or unsubscribe !). If it’s from a person, then let them know you have changed your mail address.

## Step 9 : Actually close your Gmail account

Once you are not receiving any more mail to your Gmail address (or barely) you can take the big step and close your Gmail account.

You can follow [Google's guide](https://support.Google.com/accounts/answer/61177) to do so. Some nice facts are that your address may never be used by anyone in the future, and that your Google Account will not be deleted in the same time.

I'm a cheater : I have not actually gone through this step yet. I think I will wait at least one year in order to be sure I'm not receiving any import mail to my Gmail address.

## Step 10 : Go the extra mile

While you are at it you may want to try and replace other Google services.

Natural followers are the Calendar and Contacts apps. These are two pieces of sensitive data that you can very easily export from Google and import into your new Fastmail account. Then you can again replace the Google Calendar and Google Contacts apps with alternatives. The Apple ones have proven far good enough for my usage.

If you want to go further, you can check the [No More Google list](https://nomoregoogle.com/) to find popular alternatives to Google Services.

{% image "./nomoregoogle.png", "no more google" %}

I was very surprised how easy it was to stop using Google Chrome. The new Firefox is fast, almost bug free, and has all the web development tools I need. I was also using Chrome on my iPhone but I like Safari more now that I have switched. The transition was smooth and intuitive and it’s obviously better integrated with iOS.

The only Google apps I really could not stop using are Maps and Docs. They both have features I could not find anywhere else and which I don't want to stop using altogether.

I hope this helps, let me know if you switch!

*You can comment this article on [Hacker News](https://news.ycombinator.com/item?id=18627509)*
