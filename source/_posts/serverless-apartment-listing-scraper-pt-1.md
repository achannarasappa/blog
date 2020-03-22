---
title: Serverless apartment web scraper with NodeJS, AWS Lambda, and Locust - Part 1/3
date: 2019-12-22 10:32:22
tags:
---
New York's apartment rental market is competitive with rentals in desireable neighborhoods being rented quickly. Let's build a Craigslist apartment listing web scraper to understand the market better and make a data driven decision on where to move.

Let's focus on this aspect of the of the apartment rental market:

**What areas in New York are most popular, have the best public transit connectivity, and offer the best amenities for their asking price?**

This will be the first of a three part series:
1. Gathering rental market data - Building a web scraper 
2. Gathering rental market data - Deploying and operating the web scraper
3. Deriving rental market insights - Analyzing the data

## Solution Space

While there are a number of different tools that can be used for web data extraction, let's impose some criteria for this project to help refine solution selection.
1. Minimize infrastructure costs (idle + active)
1. Horizontally scalability of data extraction
1. Maintainability of data extraction logic

### Technologies

The solution space of web data extraction is quite crowded with a number of open source projects and commercial offerings. In this case we will use:
* **AWS RDS** (storage)
* **AWS Lambda** (compute)
* **NodeJS** (runtime)
* [**Locust**](https://locust.dev) (scraping framework)

Disclosure: Locust is developed by me

### Approach

First, we'll divide the web scraping problem into a more manageable sub-problems:

1. Understand site and page structure
    * How to pages relate to one another?
    * Which pages contain relevant information?
    * What data attributes are useful for this problem?
    * Is any processing needed to clean up or restructure the data?
1. Configuring the web scraper
    * When should the scraper stop gathering listings?
    * How can we gather data quickly while being considerate of site load?
    * How should we handle error conditions?
1. Persisting data
    * How do the entities we store relate to one another?
    * How do we structure the data we store?
    * Should raw output or cleaned/formatted data be stored?
1. Deployment and infrastructure on AWS
    * What infrastructure do we need to provision on AWS?

### Assumptions

We'll also need to validate some assumptions during initial discovery and as we begin capturing data:

1. Site and page structure
    1. There are only two types of pages - indexes and details
    1. There is only one page structure for each type of entity with minor variations
1. Site and user behaviors
    1. When listings are removed or retired, the unit is taken by a new tenant

## Discovery

### Page categorization

Starting by visiting the [CL New York page apartment listing page](https://newyork.craigslist.org/search/apa?s=120) and exploring, there's ostensibly only two relevant groupings of pages each with different types of information we need to extract:

1. **Entity index** - list of multiple entities with some limited detail
    ![entity index](https://thepracticaldev.s3.amazonaws.com/i/i3snzq7whmzlgurkngbj.png)
1. **Entity detail** - detailed information on a single entity
    ![entity detail](https://thepracticaldev.s3.amazonaws.com/i/txyfjfz3ro2kuyj2much.png)

### Page relationships

Web pages are linked to one another with anchor elements (`<a>` tags). The `href` attributes of these elements link to other related pages and  can be used to crawl the entirety of the site. Since we're only interested in the above two type of entities, the only links we are interested in are those to other entities.

To get an idea of what links are on an entity index and entity detail page, `$$('a').map(el => el.href)` can be run in Chrome Developer Tools.

![links on page](https://thepracticaldev.s3.amazonaws.com/i/xmfk3s3qw5bpdzqsoxwj.png)

Here, there are 350+ links from this page which are mostly not relevant or duplicates. However through examining the results, we find that there are two link patterns that correspond to the two types of entities identified above:

1. Entity index - `https://newyork.craigslist.org/search/apa?s=<page offset>`
1. Entity detail - `https://newyork.craigslist.org/<region>/apa/d/<listing name>/<listing id>.html`

The scraper will need to bound it's crawl of the site to these two types of pages.

### Entity attributes

In the previous step, we've already identified links as one of the data attributes that need to be extracted to crawl a site. Since the entity information on an entity index page is rather limited, we'll focus on extracting entity attributes from the entity detail page.

Since it's not yet clear at this stage, what listing elements influence apartment popularity, let's capture as many attributes as possible and cleave away irrelevant attributes at a later time.

Below are some attributes and their corresponding locations on the page to capture as a first pass:

![page attributes](https://thepracticaldev.s3.amazonaws.com/i/v5sk1s5gt807a0f36s31.png)

* title
* price
* bedroom_count
* size
* attributes
* latitude
* longitude

For each of these, we'll need to find the [CSS selectors](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors). In some cases, (e.g. `bedroom_count`) we'll need to capture the an element that contains the data attributes value and use regular expressions later on to process the data and extract the information needed.

### Summary

At this point, we have enough understanding of the site to start writing code / configuration. Before moving on from discovery, let's summarize what we've learned about the site:

* There are two types of pages that have data we're interested in:
    1. **Entity index** - list of multiple entities with some limited detail
        * **Information to extract**: links to other entity indexes and entity detail pages
        * **Transforms** - filtering out links to extraneous pages that are not entity indexes or entity detail pages
        * **Outputs** - list of links to entity index and entity detail pages that should be fed back into the web scraper to scrape next
    1. **Entity detail** - detailed information on a single entity
        * **Information to extract** - attributes of the single entity
        * **Transforms** - formatting, cleaning, or restructuring entity attributes
        * **Outputs** - a single entity to persist to a datastore

## Execution

### Setup

Refer to the [setup section](https://github.com/achannarasappa/locust-examples/tree/master/apartment-listings#setup) in the example repo for instructions on how to setup the required tools and dependencies to run the subsequent steps locally.

### Approach

The high level process flow will look something like this:
![process flow](https://thepracticaldev.s3.amazonaws.com/i/zbjhxqxkaya9bgjzddyz.png)

Locust will handle the labeled scraping and queueing steps with the right job configuration file. The only logic that needs to be developed is the integration with the persistence layer.

Steps 3, 4, and 5 will loop until a stop condition (step 6) is met at which point the crawl will end.

### Defining the job

We'll start by defining some base properties for the job that will govern how it will operate. We'll choose some reasonable starting values for these and work to refine them as we learn more about the site behaviors and limitations.

* Entrypoint - As is standard for web crawlers, an entrypoint url defines the first page that is crawled and where links to subsequent pages is extracted. A good starting url will link to other relevant pages and in this case, that would be the first entity index page `https://newyork.craigslist.org/search/apa`.
* Stop Conditions - When should the job stop? As a starting point, we'll set a depth limit of 2 indicating that the job shouldn't crawl pages that are more than two degrees of separation from the entrypoint page.
* Throttling - How should we limit the web crawler so it does not put too great a load on the site? Many servers will enforce rate limitations and ban clients that exceed those limitations. We need to define some starting limitations for the crawler to obey so as to not come up against these limitations. We can start with two concurrent job at any given time and introduce a delay of 3000ms before each job.


Below is a [Locust job definition](https://locust.dev/docs/api#object-jobdefinition) that captures that above:

```js
// job.js
module.exports = {
  url: 'https://newyork.craigslist.org/search/apa', // entrypoint url where the job start
  config: {
    name: 'apartment-listings',
    concurrencyLimit: 2, // maximum concurrent number of jobs
    depthLimit: 2, // maximum link distance of a page from the entrypoint url to be scraped
    delay: 3000, // delay in milliseconds before starting a scrape job
  },
  connection: {
    redis: { // locust queue connection details
      port: 6379,
      host: 'localhost'
    },
    chrome: { // locust chrome connection details
      browserWSEndpoint: 'ws://localhost:3000',
    },
  },
  start: () => null,
};
```

Note: Locust's CLI tool can be used to interactively generate this file with [`locust generate`](https://locust.dev/docs/develop#create-a-job)

Next, let's test that this job works with [`locust run job.js`](https://locust.dev/docs/develop#run):
```sh
❯ locust run job.js -l
Running in single job mode. Queue related hooks and configuration will be ignored. Check docs for more information.
response:
  ok:         true
  status:     200
  statusText: OK
  headers:
    last-modified:             Sat, 30 Nov 2019 17:26:56 GMT
    cache-control:             max-age=900, public
    date:                      Sat, 30 Nov 2019 17:26:55 GMT
    content-encoding:          gzip
    vary:                      Accept-Encoding
    content-length:            36348
    content-type:              text/html; charset=utf-8
    x-frame-options:           SAMEORIGIN                                                           
    server:                    Apache
    expires:                   Sat, 30 Nov 2019 17:41:56 GMT
    set-cookie:                cl_b=4|c67de625ad2525f94f6b813ca1498758bbff6f5a|1575135224cQqUI;path=/;domain=.craigslist.org;expires=Fri, 01-Jan-2038 00:00:00 GMT
    strict-transport-security: max-age=86400
  url:        https://newyork.craigslist.org/search/apa
links:
  - https://newyork.craigslist.org/
  - https://newyork.craigslist.org/
  - https://post.craigslist.org/c/nyc
  - https://accounts.craigslist.org/login/home
  - https://newyork.craigslist.org/search/apa#
  - https://newyork.craigslist.org/search/apa#
  ... 
```

Here again we see the ~350 links. Next let's strip out links to pages that are not relevant.

### Filtering links

In order to filter the links down to just entity index and detail pages, we can apply a [filter function](https://locust.dev/docs/api#function-filter) with a couple regular expressions. Referring back to the two page patterns identified as relevant earlier, these can be converted into regular expressions to bound the pages the job run on.

```js
// job.js
const isDetailUrl = (url) => /newyork\.craigslist\.org\/(.*)\/?apa\/d\/(.*)\.html(?<!#)$/.test(url);
const isIndexUrl = (url) => /newyork\.craigslist\.org\/search\/apa\?s=([0-9]*)$/.test(url);

module.exports = {
  // ...
  filter: (links) => links.filter(link => isIndexUrl(link) || isDetailUrl(link)),
  // ...
};
```

Running `locust run job.js -l` again will yield a much less noisy set of links. We still see duplicates however these will be filtered out internally by Locust.

### Extracting data

Using upon the page elements identified earlier, we can add an [extract function](https://locust.dev/docs/api#function-extract) to define entity attributes to extract from the page for our job. We'll also need to handle cases when an element at a selector does not exist since we have two page structures that need to be handled.

```js
// job.js
module.exports = {
  // ...
  extract: async ($, page) => ({
    'title': await $('.postingtitletext #titletextonly'),
    'price': await $('.postingtitletext .price'),
    'housing': await $('.postingtitletext .housing'),
    'location': await $('.postingtitletext small'),
  }),
  // ...
};
```

Here, the `$` convenience function selects the [text content](https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent) of the first element the CSS selector matches.

We also want to extract out the listing attributes which correspond to multiple HTML elements with attributes we're interested in. Locuts' `$` is design to only extract a single element from the page so we'll need to use Puppeteer's version of [Document.querySelectorAll](https://developer.mozilla.org/en-US/docs/Web/API/Document/querySelectorAll), [page.$$eval](https://pptr.dev/#?product=Puppeteer&version=v1.18.1&show=api-pageevalselector-pagefunction-args) to extract multiple attributes:

```js
// job.js
module.exports = {
  ...
  extract: async ($, page) => ({
    ...
    'images': await page.$$eval('#thumbs .thumb', (elements) => elements.map((el) => el.getAttribute('href'))).catch(() => null),
    ...
  }),
  ...
};
```

Applying the same approach to the other entity attributes identified earlier, we will end up with an extract function that looks something like [this](https://github.com/achannarasappa/locust-examples/blob/master/apartment-listings/src/job.js#L72):

Again running this with Locust CLI returns the unformatted data that we expect:

```sh
❯ locust run job.js   
Running in single job mode. Queue related hooks and configuration will be ignored. Check docs for more information.
data: 
  title:            Great Location 1 Bd Kent Ave
  price:            $1995
  housing:          / 1br - 550ft2 - 
  location:          (Bed Sty/ Clinton Hill)
  datetime:         2019-11-30T09:18:35-0500
  images: 
    - https://images.craigslist.org/00n0n_4f3tg9LaeXL_600x450.jpg
    - https://images.craigslist.org/00202_6CW2GEUYqb5_600x450.jpg
    - https://images.craigslist.org/01313_dP3ybMPhO0j_600x450.jpg
    - https://images.craigslist.org/00909_71bNJzxnYCJ_600x450.jpg
    - https://images.craigslist.org/00606_aJQr6Xo6hFU_600x450.jpg
    - https://images.craigslist.org/00C0C_9dQLT85mc4e_600x450.jpg
    - https://images.craigslist.org/00Y0Y_b1LXFSOQtEH_600x450.jpg
  attributes: 
    - application fee details: $20 credit check
    - broker fee details: one month
    - cats are OK - purrr
    - apartment
    - laundry in bldg
    - listed by: Lawrence Amrhein/Exit All Seasons
  google_maps_link: https://www.google.com/maps/preview/@40.694989,-73.959472,16z
url:      https://newyork.craigslist.org/brk/apa/d/brooklyn-great-location-1-bd-kent-ave/7029456524.html
```

Looking at a few of the attributes, all the off the data is present but not in a fully usable state (e.g. housing). Next, we'll setup some transformations to clean up the data before we persist it.

### Transforming data

Some of the data that the page exposes can be used as is however there some attributes that we want to clean, transform, or split. Below are the attributes that we'll seek to pull from the raw output:

* price - parse into numerical value with two decimal places
* bedroom count - parse number followed by `br` from `housing` field
* size - parse number followed by `ft2` from `housing` field
* latitude - parse string from `google_maps_link`
* longitude - parse string from `google_maps_link`
* date_posted - parse ISO 8601 datetime from human readable datetime

That transform function would look like this:
```js
// job.js
const moment = require('moment')

// ...

const transformListing = (listing) => ({
  title: listing.title,
  price: parseInt(((listing.price || '').match(/\$([0-9]*)/) || [])[1] || 0, 10),
  location: matchObjectPropertyRegexOrNull(listing, 'location', /\((.*)\)/),
  bedroom_count: matchObjectPropertyRegexOrNull(listing, 'housing', /([0-9]*)br/),
  size: matchObjectPropertyRegexOrNull(listing, 'housing', /([0-9]*)ft2/),
  date_posted: listing.datetime ? moment(listing.datetime).format('YYYY-MM-DD HH:mm:ss') : null,
  attributes: listing.attributes || [],
  images: listing.images || [],
  description: listing.description,
  latitude: matchObjectPropertyRegexOrNull(listing, 'google_maps_link', /@([0-9.-]*),/),
  longitude: matchObjectPropertyRegexOrNull(listing, 'google_maps_link', /,([0-9.-]*),/),
});

const matchObjectPropertyRegexOrNull = (object, property, regex) => {

  if (!object[property])
    return null;

  if (!object[property].match(regex))
    return null;

  return object[property].match(regex)[1]

}

module.exports = {
  extract: async ($, page) => transformListing({
    // ...
  }),
  // ...
};
```

Layering the transform function into the job definition file and running with the CLI, the output should include the transformed output:
```sh
❯ locust run ./apartment-listings/src/job.js
Running in single job mode. Queue related hooks and configuration will be ignored. Check docs for more information.
data: 
  title:         Great Location 1 Bd Kent Ave
  price:         1995
  location:      Bed Sty/ Clinton Hill
  bedroom_count: 1
  size:          550
  date_posted:   2019-11-30 09:18:35
  attributes: 
    - application fee details: $20 credit check
    - broker fee details: one month
    - cats are OK - purrr
    - apartment
    - laundry in bldg
    - listed by: Lawrence Amrhein/Exit All Seasons
  images: 
    - https://images.craigslist.org/00n0n_4f3tg9LaeXL_600x450.jpg
    - https://images.craigslist.org/00202_6CW2GEUYqb5_600x450.jpg
    - https://images.craigslist.org/01313_dP3ybMPhO0j_600x450.jpg
    - https://images.craigslist.org/00909_71bNJzxnYCJ_600x450.jpg
    - https://images.craigslist.org/00606_aJQr6Xo6hFU_600x450.jpg
    - https://images.craigslist.org/00C0C_9dQLT85mc4e_600x450.jpg
    - https://images.craigslist.org/00Y0Y_b1LXFSOQtEH_600x450.jpg
  latitude:      40.694989
  longitude:     -73.959472
url:      https://newyork.craigslist.org/brk/apa/d/brooklyn-great-location-1-bd-kent-ave/7029456524.html
```

With the right data attributes, the next step is to start persisting the data.

### Persisting data

Since the attributes and structure of listing data is consistent for the most part, a relational database is a suitable storage solution. 

#### Postgres Setup

Let's proceed with starting up a local Postgres server:
```sh
docker run -it -p 5432:5432 --name listings-pg postgres:10
```

Then creating a Postgres Schema and table with schema matching the transformed data structure:
```sql
CREATE SCHEMA listing;

CREATE TABLE listing.home (
    id integer NOT NULL,
    title character varying,
    price numeric,
    location character varying,
    bedroom_count numeric,
    size character varying,
    date_posted timestamp with time zone,
    attributes jsonb,
    images jsonb,
    description character varying,
    latitude character varying,
    longitude character varying
);
```

With the Postgres database setup with the proper schema, the next step is to update the job to insert listings.

#### Updating the job

In order to insert a new listing after each job run, a postgres client will be needed and the popular [`pg` library](https://github.com/brianc/node-postgres) will work.

In the job file, a connection will also need to be established for each job run since all jobs run in independent AWS Lambda functions along with a call to execute an `INSERT` query:

```js
// job.js
const { Client } = require('pg')

// ...

const saveListing = async (listing) => {

  const client = new Client({
    host: 'localhost',
    database: 'postgres',
    user: 'postgres',
    password: 'postgres',
    port: 5432,
  })
  await client
    .connect();
  await client.query({
    text: [
      'INSERT INTO listing.home',
      '(title, price, "location", bedroom_count, "size", date_posted, "attributes", images, description, latitude, longitude)',
      'VALUES(',
      '$1,',
      '$2,',
      '$3,',
      '$4,',
      '$5,',
      '$6,',
      '$7,',
      '$8,',
      '$9,',
      '$10,',
      '$11',
      ');',
    ].join(' \n'),
    values: Object.values(listing),
  }, () => {
    client.end()
  });

};
```

Then, a Locust [`after` hook](https://locust.dev/docs/api#function-start) will need to be added to the job definition file in which the `saveListing` function will be called after scraping the site and transforming the output data.

`saveListing` should also only be called on the entity detail pages and not on the entity index pages so a conditional is in order:

```js
// job.js
module.exports = {
  // ...
  after: async (jobResult, snapshot, stop) => {

    // defined earlier for the filter function
    if (isListingUrl(jobResult.response.url)) {

      await saveListing(jobResult.data)

    }

    return;

  },
  // ...
};
```

With the integration of the persistence layer, the job definition is for the most part complete. The next step is to do a test run of the job locally before deploying to AWS.

The complete job definition file can be found in [the example repo](https://github.com/achannarasappa/locust-examples/blob/master/apartment-listings/src/job.js).

### Putting it all together

Earlier, `locust run` was used to scrape a single page to validate that the `extract` function worked as expected with the queue related features of Locust disabled. Before going through the trouble of setting up infrastructure on AWS and pushing the job up, it is best to run the the job locally with [`locust start`](https://locust.dev/docs/develop#run). This will run the job very similarly to how it will operate on AWS Lambda (or any cloud provider). This will also run a CLI UI that shows active jobs, their status, and queue information which is useful to tracking job progress and uncovering issues with the job.

First, ensure that dependent systems are up (postgres, redis, chrome) from [this docker-compose.yml](https://github.com/achannarasappa/locust-examples/blob/master/apartment-listings/docker-compose.yml) file and start them if not with `docker-compose up`

Next, run the start command with the job file and monitor it's progress:
```sh
locust start ./job.js
```
![monitor run](https://thepracticaldev.s3.amazonaws.com/i/nroko6ie4gb8kxzomg23.png)

Connecting to the Postgres database and `SELECT`ing contents of the `listing.home` table, we can observe new listings being added while the job is running:
![postgres](https://thepracticaldev.s3.amazonaws.com/i/r0kyy9d2srjs4fq2le62.png)

This is a good indication that the job is stable and is suitable to push up to AWS.

Up until this point, the we've hardcoded configuration for local runs in the job definition file. Before pushing up to AWS, AWS-specific integrations will need to be added including environment variables and a Locust [`start` hook](https://locust.dev/docs/api#function-start) to define for Locust how to invoke a new Lambda instance on AWS.

## What's next

In part two, we'll deploy the scraper to AWS and begin gathering data.