---
title: Incremental crawler with Scrapy and MongoDB
date: 2018-12-17
categories: en
tags: dev scraping
summary: Tutorial for crawling a website incrementally. Each new scraping session will only scrape new items.
---

- updated on 25/12/2018 : [*fixed from_crawler method overriding*](https://github.com/adipasquale/blog.dipasquale.fr/commit/8d2b191e1a1a7151c6b01b088d9c98812376aec1)

In this post I will show you how to scrape a website incrementally.
Each new scraping session will only scrape new items.
We will be crawling [Techcrunch blog posts](https://techcrunch.com/) as an example here.

This tutorial will use Scrapy, a great Python scraping library.
It's simple yet very powerful.
If you don't know it, have a look at their [overview page](https://doc.scrapy.org/en/latest/intro/overview.html).

We will also use MongoDB, the famous NoSQL DB, but it would be a similar process with any DB you want.

**TLDR;** if you already know Scrapy, head to the [last part about incremental scraping](#incremental). You can find the full code for this project [here on GitHub](https://github.com/adipasquale/techcrunch-incremental-scrapy-spider-with-mongodb).

# Setup your Scrapy spider

Start by installing Scrapy

```sh
pip3 install scrapy
```

*(in a real project, you would use a [virtualenv](https://virtualenv.pypa.io/en/latest/) and a `requirements.txt` file)*

and initialize your project with:

```sh
scrapy startproject tc_scraper
cd tc_scraper
scrapy genspider techcrunch techcrunch.com
```

# Scrape the posts

## Play around in the shell

First have a look at the DOM structure on [https://www.techcrunch.com](https://www.techcrunch.com), using your browser's developer tools.

**Make sure to disable Javascript**, because the scraper will not execute it by default.
It's doable with Scrapy, but it's not the point of this tutorial.
I'm using [this extension](https://addons.mozilla.org/en-US/firefox/addon/javascript-toggler/) to easily disable JS on Firefox.

{% image "./techcrunch-inspect.png", "inspection of Techcrunch's DOM in Firefox" %}


You can then open a Scrapy shell with

```sh
scrapy shell https://www.techcrunch.com
```

This shell is very helpful to play around and figure out how to extract the data. Here are some commands you can try one by one:

```py
response.css(".post-block")
posts = response.css(".post-block")
posts[0]
posts[0].css(".post-block__title__link")
title = posts[0].css(".post-block__title__link")
title
title.css("::attr(href)").extract()
title.css("::attr(href)").extract_first()
```

We are using CSS selectors, and the `attr` function on CSS3 pseudo-elements.
Learn more about this extraction part in the [scrapy docs](https://doc.scrapy.org/en/latest/topics/selectors.html).

## Scrapy architecture

{% image "./scrapy_architecture.png", "scrapy architecture" %}

This diagram  from [scrapy docs](https://doc.scrapy.org/en/0.10.3/topics/architecture.html) is a quick overview of how Scrapy works:
- The <span style="color:#DC2300;">Spider</span> yields <span style="color:#8AAF59">Requests</span>, which are sent to the <span style="color:#CCCCFF">Scheduler</span>.
- The <span style="color:#CCCCFF">Scheduler</span> sends <span style="color:#8AAF59">Requests</span> to the <span style="color:#E6E64C">Downloader</span>, which executes them against the distant website.
- The <span style="color:#8AAF59">Responses</span> are sent to the <span style="color:#DC2300;">Spider</span> for parsing.
- The <span style="color:#DC2300;">Spider</span> parses and yields Items, which are sent to the <span style="color:#FF9966">Item Pipeline</span>.
- The <span style="color:#FF9966">Item Pipeline</span> is responsible for processing them and storing them.

In this tutorial, we will not touch the <span style="color:#CCCCFF">Scheduler</span>, nor the <span style="color:#E6E64C">Downloader</span>.

We will only write a <span style="color:#DC2300;">Spider</span> and tweak the <span style="color:#FF9966">Item Pipeline</span>.

## Scrape the list pages

So let's write the first part of the scraper:

```py
# spiders/techcrunch.py

import scrapy


class TechcrunchSpider(scrapy.Spider):
    name = 'techcrunch'
    allowed_domains = ['techcrunch.com']
    start_urls = ['https://techcrunch.com/']

    def parse(self, response):
        for post in response.css(".post-block"):
            title = post.css(".post-block__title__link")
            url = title.css("::attr(href)").extract_first()
            yield scrapy.Request(url, callback=self.parse_post)
        if response.css(".load-more"):
            next_page_url = response.css(".load-more::attr(href)").extract_first()
            yield scrapy.Request(next_page_url)

    def parse_post(self, response):
        pass
```

Here is a walkthrough of this spider:

- It starts by scraping the start url https://techcrunch.com/
- It goes through all posts blocks, extracts the link, and enqueues a new request.
This new request will use a different callback from the default one: `parse_post`
- After going through all the posts, it looks for a next page link, and if it finds it, it enqueues a new request with that link.
This request will use the default callback `parse`

You can run it with `scrapy crawl techcrunch`, but be aware that it will go through ALL the pages (thousands here), so be ready to hit `CTRL+C` to stop it!

## Add a pages limit argument

In order to avoid this problem, let's add a pages limit argument to our spider right now:

```py
# spiders/techcrunch.py

import scrapy
import re


class TechcrunchSpider(scrapy.Spider):
    # ...

    def __init__(self, limit_pages=None, *args, **kwargs):
        super(TechcrunchSpider, self).__init__(*args, **kwargs)
        if limit_pages is not None:
            self.limit_pages = int(limit_pages)
        else:
            self.limit_pages = 0

    def parse(self, response):
        # ...
        if response.css(".load-more"):
            next_page_url = response.css(".load-more::attr(href)").extract_first()
            # urls look like https://techcrunch.com/page/4/
            match = re.match(r".*\/page\/(\d+)\/", next_page_url)
            next_page_number = int(match.groups()[0])
            if next_page_number <= self.limit_pages:
                yield scrapy.Request(next_page_url)
    # ...
```

In the constructor, we allow passing a new kwarg called limit_pages, which we cast to an integer.
In the `parse` method, we extract the next page number thanks to a regex on the url. Then we compare it to the `limit_pages` argument, and only if it's below, we enqueue the next page request.

You can now run the spider safely with:

```sh
scrapy crawl techcrunch -a limit_pages=2
```

## Scrape the post pages

So far, we have left the `scrape_post` request empty, so our spider is not actually scraping anything.
Here is what post pages look like (again, without JS):

{% image "./techcrunch-post.png", "a Techcrunch post page" %}

Before writing the scraper method, we need to declare the items that we are going to scrape:

```py
# items.py
import scrapy

class BlogPost(scrapy.Item):
    url = scrapy.Field()
    title = scrapy.Field()
    author = scrapy.Field()
    content = scrapy.Field()
    published_at = scrapy.Field()
```

*Note: There is now a [simple way](https://stackoverflow.com/a/5077350) to have a dynamic schema without manually declaring all the fields.
I will show how to use it in a next article*

You can open a new Scrapy shell to play around on any post page and figure out the selectors you are going to use.
For example:

```sh
scrapy shell https://techcrunch.com/2017/05/01/awesomeness-is-launching-a-news-division-aimed-at-gen-z/
```

And once you have figured them out, you can write the scraper's missing method:

```py
# spiders/techcrunch.py

from tc_scraper.items import BlogPost
import datetime


class TechcrunchSpider(scrapy.Spider):

    # ...

    def parse_post(self, response):
        item = BlogPost(
            title=response.css("h1::text").extract_first(),
            author=response.css(".article__byline>a::text").extract_first().strip(),
            published_at=self.extract_post_date(response),
            content=self.extract_content(response),
            url=response.url
        )
        yield(item)

    def extract_post_date(self, response):
        date_text = response.css("meta[name='sailthru.date']::attr(content)").extract_first()
        return datetime.datetime.strptime(date_text, "%Y-%m-%d %H:%M:%S" )

    def extract_content(self, response):
        paragraphs_texts = [
        p.css(" ::text").extract()
            for p in response.css(".article-content>p")
        ]
        paragraphs = ["".join(p) for p in paragraphs_texts]
        paragraphs = [re.subn("\n", "", p)[0] for p in paragraphs]
        paragraphs = [p for p in paragraphs if p.strip() != ""]
        return "\n\n".join(paragraphs)
```

So what's going here?

- We instanciate a `BlogPost` item that we then yield, that's the Scrapy way.
- Most of the fields are straightforward, so we write their selectors inline.
Some others are more complicated so we have extracted them to independent methods.
- The `published_at` field is a bit tricky because in the visible DOM there is no plain text datetime, only a vague 'X hours/days ago'.
If you inspect the DOM closely, you will find this meta `sailthru.date` that is easy to use and parse.
- The `extract_content` method is quite involved, but it's really not key to this article's objective.
We are basically joining the texts from the different paragraphs in a way that's human readable.
Because the content is actually HTML, we are losing some infos in the process, like the links and images.

You can now run the spider with:

```sh
scrapy crawl techcrunch -a limit_pages=2 -o posts.json
```

and Scrapy will generate a nice `posts.json` file with all the scraped items. Yay!

{% image "./cat_posts_json.png", "display posts.json contents" %}

# <a name="incremental"></a>Incremental Scraping

## Store items in MongoDB

So in the last step we exported the items to a JSON file.
For long term storage and re-use, it's more convenient to use a database.
We will use MongoDB here, but you could use a regular SQL database too.
I personally find it convenient on scraping projects to use a NoSQL database because of the frequent schema changes, especially as you initially iterate on the scraper.

If you are on Mac OS X, these commands will install MongoDB server and start it as a service:

```sh
brew install mongodb
brew services start mongodb
```

We are going to create a Scrapy pipeline so that each yielded item will get saved to MongoDB.
This process is well documented in [Scrapy's docs](https://doc.scrapy.org/en/latest/topics/item-pipeline.html?highlight=mongo#write-items-to-mongodb).

```py
# pipelines.py

import pymongo


class MongoPipeline(object):

    collection_name = 'tc_posts'

    def __init__(self, mongo_uri, mongo_db):
        self.mongo_uri = mongo_uri
        self.mongo_db = mongo_db

    @classmethod
    def from_crawler(cls, crawler):
        return cls(
            mongo_uri=crawler.settings.get('MONGO_URI'),
            mongo_db=crawler.settings.get('MONGO_DATABASE', 'tc_scraper')
        )

    def open_spider(self, spider):
        self.client = pymongo.MongoClient(self.mongo_uri)
        self.db = self.client[self.mongo_db]

    def close_spider(self, spider):
        self.client.close()

    def process_item(self, item, spider):
        self.db[self.collection_name].insert_one(dict(item))
        return item
```

and activate it in the settings:

```py
# settings.py

ITEM_PIPELINES = {
   'tc_scraper.pipelines.MongoPipeline': 300,
}
```

Last step is to install the new `pymongo` dependency:

```sh
pip3 install pymongo
```

You can now re-run the spider:

```
scrapy crawl techcrunch -a limit_pages=2
```

Feel free to open a mongo shell and check that the items were indeed inserted:

```sh
mongo localhost/tc_scraper
> db.tc_posts.count()
40
> db.tc_posts.findOne()
{
	"_id": ObjectId("5c09294a7fa9c70f84e43322"),
	"url": "https://techcrunch.com/2018/12/06/looker-looks-to-future-with-103m-investment-on-1-6b-valuation/",
	"author": "Ron Miller",

  ...
```

## Update existing items

If you run the spider again, you will notice that you now have 80 items in your database.
Let's update the Pipeline so that it does not insert a new post each time, but rather updates the existing one if it already exists.

```py
# pipelines.py

...

    def process_item(self, item, spider):
        self.db[self.collection_name].find_one_and_update(
            {"url": item["url"]},
            {"$set": dict(item)},
            upsert=True
        )
        return item
```

Here we are using the url as the key, because unfortunately there does not seem to be a more canonical ID in the DOM.
It should work as long as Techcrunch does not change their posts slugs or published dates often.

You can drop all items and re-run the spider twice:

```sh
echo "db.tc_posts.drop();" | mongo localhost/tc_scraper
scrapy crawl techcrunch -a limit_pages=2
scrapy crawl techcrunch -a limit_pages=2
echo "db.tc_posts.count();" | mongo localhost/tc_scraper
```

You should now have only 40 items in the database.

*Note: If you are scared about running so many requests against Techcrunch.com, be aware that by default, Scrapy will use a [cache](https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#module-scrapy.downloadermiddlewares.httpcache).
For most requests, it will not re-run them every single time, but instead re-use the previous response*

## Limit crawls to new items

So far our solution is almost complete, but it will re-scrape all items every time you start it.
In this step, we will make sure that we don't re-scrape items uselessly, in order to have faster scraping sessions, and to limit our requests rate to the website.

What we will do is to update the spider so that it prevents requests to items that were already scraped before and are in the database.

First let's extract the MongoDB connection logic from the pipeline in order to re-use it in the spider:

```py
# mongo_provider.py (new file)

import pymongo


class MongoProvider(object):

    collection_name = 'tc_posts'

    def __init__(self, uri, database):
        self.mongo_uri = uri
        self.mongo_db = database or 'tc_scraper'

    def get_collection(self):
        self.client = pymongo.MongoClient(self.mongo_uri)
        return self.client[self.mongo_db][self.collection_name]

    def close_connection(self):
        self.client.close()
```

and update the pipeline accordingly:

```py
# pipelines.py

from tc_scraper.mongo_provider import MongoProvider


class MongoPipeline(object):

    def __init__(self, settings):
        self.mongo_provider = MongoProvider(
            settings.get('MONGO_URI'),
            settings.get('MONGO_DATABASE')
        )

    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler.settings)

    def open_spider(self, spider):
        self.collection = self.mongo_provider.get_collection()

    def close_spider(self, spider):
        self.mongo_provider.close_connection()

    def process_item(self, item, spider):
        self.collection.find_one_and_update(
            {"url": item["url"]},
            {"$set": dict(item)},
            upsert=True
        )
        return item
```

*You can re-run the scraper at this checkpoint, nothing should have changed*

We can now update the spider so that it uses this mongo provider:

```py
# spiders/techcrunch.py

...
from tc_scraper.mongo_provider import MongoProvider


class TechcrunchSpider(scrapy.Spider):
    ...
    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        kwargs['mongo_uri'] = crawler.settings.get("MONGO_URI")
        kwargs['mongo_database'] = crawler.settings.get('MONGO_DATABASE')
        return super(TechcrunchSpider, cls).from_crawler(crawler, *args, **kwargs)

    def __init__(self, limit_pages=None, mongo_uri=None, mongo_database=None, *args, **kwargs):
        ...
        self.mongo_provider = MongoProvider(mongo_uri, mongo_database)
        self.collection = self.mongo_provider.get_collection()
        last_items = self.collection.find().sort("published_at", -1).limit(1)
        self.last_scraped_url = last_items[0]["url"] if last_items.count() else None

    def parse(self, response):
        for post in response.css(".post-block"):
            ...
            if url == self.last_scraped_url:
                print("reached last item scraped, breaking loop")
                return
            else:
                yield scrapy.Request(url, callback=self.parse_post)
```

Here is a quick breakdown of what we are doing here:

- In order to use the spider's settings that contain the mongo credentials, we need to do override this `from_crawler` method. It's quite verbose, and not very intuitive. I agree it's annoying.
- We then use these settings in the constructor to initialize a MongoDB connection thanks to our new `MongoProvider` class.
- We query Mongo for the last scraped item and store it's url.
Here we are sorting the posts on the `published_at` descendingly, we made sure that this is consistent with Techcrunch's sorting, or else our algorithm would not work properly.
- In the parsing loop, we break and stop the scraping as soon as we reach a post with this url.
We do it by preventing yielding the request, and breaking from the loop.

You can now perform a few tests, drop some of the last items from MongoDB and re-scrape, you will see that it only scrapes the missing items and stops. Success!

```sh
mongo localhost/tc_scraper
> last_item = db.tc_posts.find().sort({published_at: -1})[0]
> db.tc_posts.remove({_id: last_item["_id"]})
> exit
scrapy crawl techcrunch -a limit_pages=2
```


# Conclusion

We now have a scraper that will do the least amount of work possible on each new run. I hope you enjoyed this tutorial, and that this gave you new ideas for scraping projects!

You can find the full code for this project here on GitHub: [adipasquale/techcrunch-incremental-scrapy-spider-with-mongodb](https://github.com/adipasquale/techcrunch-incremental-scrapy-spider-with-mongodb).

You can deploy your spider to [ScrapingHub cloud](https://scrapinghub.com/scrapy-cloud), and schedule it to run daily on their servers.
I'm not affiliated to them in any way, it's just an awesome product and their free plan already does a lot.
By the way, ScrapingHub is the main contributor to the fully open-source Scrapy project that we just used.

To go further, you can implement a new `force_rescrape` argument, that will bypass our limit and force going through all the items again.
This could be useful if you update the `scrape_post` method, or if Techcrunch changes their DOM structure.

Let me know if you use this technique in one of your projects!

[Discuss on Hacker News](https://news.ycombinator.com/item?id=18697956)
