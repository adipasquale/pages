---
title: Scraping tips
date: 2019-12-10
tags:
  - python
  - scraping
---

Scraping websites can sometimes get tricky, but that's when things gets interesting.
When you're aiming to scrape data from a website that doesn't want to be scraped, it becomes like a game of cat and mouse.

![](https://i.imgur.com/HwN6qYV.jpg)

The website tries to identify robots and prevent them from accessing the data, without impacting actual users browsing the website.
The robots are thus designed to mimick as closely as possible actual users' behaviour, so as to become stealth.

Here is a list of recommendations, more or less obvious, but always useful to remind of.


## Keep It Simple, Stupid

Websites have become complex with Single Page Applications and React, Vue...
Do not forget that browsing websites comes down to HTTP requests being sent to servers.

There are two approaches to scraping:

1. reproduce individual HTTP requests one by one
2. mimick an actual browser and user clicking around

💁🏽‍♀️ As often as you can, try to use the first method. It is usually much more reliable and efficient.

However, the two approaches are complimentary.
Sometimes the second one is necessary, if the interactions between the JS and the HTTP requests are too entangled.

## Sniff out APIs

Look for public APIs that return JSON instead of HTML that you have to parse.
Use your browser's network tab XHR filter to do so.
This is often the case with Single Page Applications.

![Product Hunt GraphQL API](https://i.imgur.com/i6erv0B.gif)

Mobile applications also often use an API to collect data from the servers.
Most of the time this API will be somehow authenticated, but it is sometimes very easy to find a working token.

⚙️ Use [Charles Proxy](https://www.charlesproxy.com/) to listen to outgoing requests from mobile applications.

## Know your HTTP basics

HTTP requests have 4 simple characteristics:

1. An URL (with optional query string params)
2. A verb (aka a method, eg: GET, POST, etc)
3. Optional Headers
4. An optional body (for non-GET requests)

The body can be any type of text or binary data, which should be described by the `Content-Type` header.
You can find every valid value [here](http://www.iana.org/assignments/media-types/media-types.xhtml)

💡 When you get stuck and cannot figure out how the website can block your request, compare these 4 characteristics to an actual request from your browser.

🔒 Also, prefer HTTPs requests, websites can get suspicious otherwise.


## Don't reinvent the wheel

I strongly recommend against using a regular HTTP library (e.g. Python's [`requests`](https://requests.readthedocs.io/) or Ruby's [`http`](https://github.com/httprb/http)) .
Instead, build upon a scraping framework like Python's [`Scrapy`](https://github.com/scrapy/scrapy), NodeJS [`headless-chrome-scraper`](https://github.com/yujiosaka/headless-chrome-crawler) or Go's [Colly](https://github.com/gocolly/colly).

You won't have to reimplement super common patterns, like requests throttling, retries policy, links traversal and deduplication, passing cookies...
As a web framework does for building websites, this will help you focus on the core specific code that targets your websites.

🙌 [`Scrapy`](https://github.com/scrapy/scrapy) is my personal favourite. It is open source with a backing company [ScrapingHub](https://scrapinghub.com/scrapy-cloud) that also provides a Heroku-like PaaS service. I'm not affiliated to them in any way, but they provide instant deployment and scheduling, with an extremely low pricing policy (it's often free).

## Become a curl Ninja

You may be tempted to use GUIs like [Postman](https://www.getpostman.com/) or [Insomnia](https://insomnia.rest/), but I would suggest you get to learn `curl` instead.
It is actually not very complicated and lets you do everything without restrictions.
You can then reproduce it in any SSH terminal.
`man curl` is your friend.

Use and abuse the "Copy as curl" feature in your network tab, to paste it in a terminal and see if it's reproductible.
What I often do is try and find the smallest set of headers and body params that still succeds by bissecting down.

*Pro Tip*: copied curl commands can be lengthy.
I often copy them in my IDE, search and replace all `-H` with `-H \n` (with regex mode activated) to get one line per header.

⚙️ [Python to curl](https://curl.trillworks.com/) is a tool that converts curl commands to Python `requests` call.

## Best practices for scrapers

- Fetch first, store the response and then parse it. This way you won't have to refetch if there is a bug in your parser.
- You can also use caching locally when developping to bypass this problem
- Distribute the workload in successive steps. eg. for a cooking website, first retrieve all recipes URLs and then fetch them instead of directly following every nested link.


## It works on my computer™️

As often, the scraper you wrote may work locally but not in production.

One source of issues is the IP of your server that may be blacklisted.
It is easy for the defending websites to block whole range of IPs that are known to come from datacenters, or to whitelist specific countries IPs.
The usual counter for this is to use proxies.
You will find countless services that sell proxies access on the internet, with varying quality.
Sometimes you won't have a choice but to use residential proxies that are costy (>100USD/mo.).

⚙️ [icanhazip](https://icanhazip.com/) is a simpleway to check the IP you're currently scraping from.

Another source of trouble is the classical environment differences that you forgot about.
For example, you may generate timestamps in your code and format them to forge requests, and the timezone may be different on your machine and on the production server.

⚙️ [httpbin](https://httpbin.org/) can be useful to double check that the received data by the distant server is actually what you meant to send.

## Have fun!

Figuring out how to bypass another developer's protections can be a really fun puzzle to solve.

🤜🤛 Be a good citizen and scrape responsibly:
- Space out requests so they don't overflow the website.
- Only scrape public data
- If you encounter private or sensitive data that should not be public, warn the website.


*Written by [Ismaël Attoumani](https://github.com/maelorn) and [Adrien Di Pasquale](https://twitter.com/hypertextadrien/), proofread by [Antoine Augusti](https://twitter.com/AntoineAugusti)*
