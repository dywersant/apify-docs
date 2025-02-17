---
title: Using proxies
description: Learn how to use and automagically rotate proxies in your scrapers by using the Apify SDK, and a bit about how to easily obtain pools of proxies.
menuWeight: 2
paths:
- anti-scraping/mitigation/using-proxies
---

# [](#using-proxies) Using proxies

In the [**Web scraping for beginners**]({{@link web_scraping_for_beginners.md}}) course, we learned about the power of the Apify SDK, and how it can streamline the development process of web crawlers. You've already seen how powerful the `apify` package is; however, what you've been exposed to thus far is only the tip of the iceberg.

Because proxies are so widely used in the scraping world, we at Apify have equipped our SDK with features which make it easy to implement them in an effective way. One of the main functionalities that comes baked into the SDK is proxy rotation, which is when each request is sent through a different proxy from a proxy pool.

## [](#implementing-proxies) Implementing proxies in a scraper

Let's borrow some scraper code from the end of the [pro-scraping]({{@link web_scraping_for_beginners/crawling/pro_scraping.md}}) lesson in our **Web Scraping for Beginners** course and paste it into a new file called **proxies.js**. This code enqueues all of the product links on [demo-webstore.apify.org](https://demo-webstore.apify.org)'s on-sale page, then makes a request to each product page and scrapes data about each one:

```JavaScript
// proxies.js
import Apify from 'apify';

await Apify.utils.purgeLocalStorage();

const requestQueue = await Apify.openRequestQueue();
await requestQueue.addRequest({
    url: 'https://demo-webstore.apify.org/search/on-sale',
    userData: {
        label: 'START',
    },
});

const crawler = new Apify.CheerioCrawler({
    requestQueue,
    handlePageFunction: async ({ $, request }) => {
        if (request.userData.label === 'START') {
            await Apify.utils.enqueueLinks({
                $,
                requestQueue,
                selector: 'a[href*="/product/"]',
                baseUrl: new URL(request.url).origin,
            });
            return;
        }

        const title = $('h3').text().trim();
        const price = $('h3 + div').text().trim();
        const description = $('div[class*="Text_body"]').text().trim();

        await Apify.pushData({
            title,
            description,
            price,
        });
    },
});

await crawler.run();
```

In order to implement a proxy pool, we will first need some proxies. We'll quickly use the free [proxy scraper](https://apify.com/mstephen190/proxy-scraper) on the Apify platform to get our hands on some quality proxies. Next, we'll need to set up a [`proxyConfiguration`](https://sdk.apify.com/docs/api/proxy-configuration#docsNav) and configure it with our custom proxies, like so:

```JavaScript
const proxyConfiguration = await Apify.createProxyConfiguration({
    proxyUrls: ['http://45.42.177.37:3128', 'http://43.128.166.24:59394', 'http://51.79.49.178:3128'],
});
```

Awesome, so there's our proxy pool! Usually, a proxy pool is much larger than this; however, a three proxie pool is total fine for tutorial purposes. Finally, we can pass the `proxyConfiguration` into our crawler's options:

```JavaScript
const crawler = new Apify.CheerioCrawler({
    proxyConfiguration,
    requestQueue,
    handlePageFunction: async ({ $, request }) => {
        if (request.userData.label === 'START') {
            await Apify.utils.enqueueLinks({
                $,
                requestQueue,
                selector: 'a[href*="/product/"]',
                baseUrl: new URL(request.url).origin,
            });
            return;
        }

        const title = $('h3').text().trim();
        const price = $('h3 + div').text().trim();
        const description = $('div[class*="Text_body"]').text().trim();

        await Apify.pushData({
            title,
            description,
            price,
        });
    },
});
```

> Note that if you run this code, it may not work, as the proxies could potentially be down at the time you are going through this course.

That's it! The crawler will now automatically rotate through the proxies we provided in the `proxyUrls` option.

## [](#debugging-proxies) A bit about debugging proxies

At the time of writing, our above scraper utilizing our custom proxy pool is working just fine. But how can we check that the scraper is for sure using the proxies we provided it, and more importantly, how can we debug proxies within our scraper? Luckily, within the same `context` object we've been destructuring `$` and `request` out of, there is a `proxyInfo` key as well. `proxyInfo` is an object which includes useful data about the proxy which was used to make the request.

```JavaScript
const crawler = new Apify.CheerioCrawler({
    proxyConfiguration,
    requestQueue,
    // Destructure "proxyInfo" from the "context" object
    handlePageFunction: async ({ $, request, proxyInfo }) => {
        // Log its value
        console.log(proxyInfo)
        // ...
        // ...
    },
});
```

After modifying your code to log `proxyInfo` to the console and running the scraper, you're going to see some logs which look like this:

![proxyInfo being logged by the scraper]({{@asset anti_scraping/mitigation/images/proxy-info-logs.webp}})

These logs confirm that our proxies are being used and rotated successfully by the Apify SDK, and can also be used to debug slow or broken proxies.

## [](#higher-level-proxy-scraping) Higher level proxy scraping

Though we will discuss it more in-depth in future courses, it is still important to mention that the Apify SDK has integrated support for [Apify Proxy](https://apify.com/proxy), which is a service that provides access to pools of both residential and datacenter IP addresses. A `proxyConfiguration` using Apify Proxy might look something like this:

```JavaScript
const proxyConfiguration = await Apify.createProxyConfiguration({
    groups: ['SHADER'],
    countryCode: 'US'
});
```

Notice that we didn't provide it a list of proxy URLs. This is because the `SHADER` group already serves as our proxy pool (courtesy of Apify Proxy).

## [](#next) Next up

[Next up]({{@link anti_scraping/mitigation/generating_fingerprints.md}}), we'll be checking out how to use two NPM packages to generate and inject [browser fingerprints]({{@link anti_scraping/techniques/fingerprinting.md}}).
