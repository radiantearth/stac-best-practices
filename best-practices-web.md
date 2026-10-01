# STAC Web Best Practices <!-- omit in toc -->

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Enable Cross-origin resource sharing (CORS)](#enable-cross-origin-resource-sharing-cors)
- [STAC on the Web](#stac-on-the-web)
  - [Schema.org, JSON-LD, DCAT, microformats, etc](#schemaorg-json-ld-dcat-microformats-etc)
  - [Deploying STAC Browser](#deploying-stac-browser)
- [Requester Pays](#requester-pays)
- [Consistent URIs](#consistent-uris)
  - [Documents available under multiple URLs](#documents-available-under-multiple-urls)

## Enable Cross-origin resource sharing (CORS)

STAC strives to make geospatial information more accessible, by putting it on the web. Fundamental to STAC's vision is that
different tools will be able to load and display public-facing STAC data. But the web runs on a [Same origin
policy](https://en.wikipedia.org/wiki/Same-origin_policy), preventing web pages from loading information from other web locations
to prevent malicious scripts from accessing sensitive data. This means that by default a web page would only be able to load STAC
[Item](https://github.com/radiantearth/stac-spec/blob/master/item-spec/item-spec.md) objects from the same server the page is on.
[Cross-origin resource sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing),
also known as 'CORS' is a protocol to enable safe communication across origins. But most web services turn it off by default. This
is generally a good thing, but unfortunately if CORS is not enabled then any browser-based STAC tool will not work.

So to enable all the great web tools (like [stacindex.org](http://stacindex.org)) to work with your STAC implementation it is essential to
'enable CORS'. Most services have good resources on how to do this, like on [AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/cors.html),
[Google Cloud Storage](https://cloud.google.com/storage/docs/cross-origin), or [Apache Server](https://enable-cors.org/server_apache.html).
Many more are listed on [enable-cors.org](https://enable-cors.org/server.html). We recommend enabling CORS for all requests ('\*'),
so that diverse online tools can access your data. If you aren't sure if your server has CORS enabled you can use
[cors-test.codehappy.dev](https://cors-test.codehappy.dev/). Enter the URL of your STAC root [Catalog](https://github.com/radiantearth/stac-spec/blob/master/catalog-spec/catalog-spec.md) or
[Collection](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md) JSON and make sure it gets a response.

## STAC on the Web

One of the primary goals of STAC is to make spatiotemporal data more accessible on the web. One would have a right to be
surprised that there is nothing about HTML in the entire specification. This is because it is difficult to specify what
should be on web pages without ending up with very bad looking pages. But the importance of having web-accessible versions
of every STAC Item is paramount.

The main recommendation is to have an HTML page for every single STAC Item, Catalog and Collection. They should be visually pleasing,
crawlable by search engines and ideally interactive. The current best practice is to use a tool in the STAC ecosystem called
[STAC Browser](https://github.com/radiantearth/stac-browser/). It can crawl most any valid STAC implementation and generate unique web
pages for each Item and Catalog (or Collection). While it has a default look and feel, the design can easily be
modified to match an existing web presence. And it will automatically turn any Item with a [Cloud Optimized
GeoTIFF](http://cogeo.org) asset into an interactive, zoomable web map (using [tiles.rdnt.io](http://tiles.rdnt.io/) to render
the tiles on a [leaflet](https://leafletjs.com/) map). It also attempts to encapsulate a number of best practices that enable
STAC Items to show up in search engines, though that part is still a work in progress - contributions to STAC Browser to help
are welcome!

Implementors are welcome to generate their own web pages, and additional tools that automatically transform STAC JSON into
html sites are encouraged. In time there will likely emerge a set of best practices from an array of tools, and we may be
able to specify in the core standard how to make the right HTML pages. But for now it is useful for STAC implementations to focus on
making data available as JSON, and then leverage tools that can evolve at the same time to make the best HTML experience. This
enables innovation on the web generation and search engine optimization to evolve independently from the core data.

### Schema.org, JSON-LD, DCAT, microformats, etc

There is a strong desire to align STAC with the various web standards for data. These include [schema.org](http://schema.org)
tags, [JSON-LD](https://json-ld.org/) (particularly for Google's [dataset
search](https://developers.google.com/search/docs/data-types/dataset)), [DCAT](https://www.w3.org/TR/vocab-dcat/)
and [microformats](http://microformats.org/wiki/about). STAC aims to work with as many as possible. Thusfar it has not seemed
to make sense to include any of them directly in the core STAC standard. They are all more intended to be a part of the HTML
pages that search engines crawl, so the logical place to do the integration is by leveraging a tool that generates HTML
from STAC like [STAC Browser](https://github.com/radiantearth/stac-browser/). STAC Browser has implemented a [mapping to
schema.org](https://github.com/radiantearth/stac-spec/issues/378) fields using JSON-LD, but the exact output is still being
refined. It is on the roadmap to add in more mapping and do more testing of search engines crawling the HTML pages.

### Deploying STAC Browser

Most public STAC implementations have a STAC Browser hosted at [stacindex.org](https://stacindex.org/catalogs).
Anyone with a public STAC implementation is welcome to have a STAC Browser instance hosted for free,
just submit it to [stacindex.org](https://stacindex.org/add).
But the stronger recommendation is to host a STAC Browser on your own domain, and to customize its
design to look and feel like your main web presence. STAC aims to be decentralized, so each STAC-compliant data catalog
should have its own location and just be part of the wider web.

## Requester Pays

It is very common that large, freely available datasets are set up with a 'requester pays' configuration. This is an option
[on AWS](https://docs.aws.amazon.com/AmazonS3/latest/userguide/RequesterPaysBuckets.html) and [on
Google Cloud](https://cloud.google.com/storage/docs/requester-pays), that enables data providers to make their data
available to everyone, while the cloud platform charges access costs
(such as per-request and data '[egress](https://www.hostdime.com/blog/data-egress-fees-cloud/)') to the user accessing the data.
For popular datasets that are large in size the egress costs can be substantial, to the point where much
less data would be available if the cost of distribution was always on the data provider.

For data providers using STAC with requester pays buckets, there are two main recommendations:

1. Put the STAC JSON in a separate bucket that is public for everyone and **not** requestor pays. This enables the STAC metadata
   to be far more crawlable and searchable, but the cost of the egress of STAC files should be miniscule compared to that of
   the actual data. The STAC community can help you work with cloud providers for potential free hosting if you are doing open
   data as requestor pays and aren't able to pay the costs of a completely open STAC bucket, as they are most all supportive of
   STAC (but no guarantees and it may be on an alternate cloud).
2. For Asset href values to resources in a requestor pays bucket, use the cloud provider-specific protocol
   (e.g., `s3://` on AWS and `gs://` on Google Cloud) instead of an `https://` url.
   Most clients do not have special handling for `https://` links to cloud provider resources that require a requestor pays flag and authentication,
   so they simply fail. Many clients have special handling for `s3://` or `gs://` URLs
   that will add a requestor pays parameter and will apply appropriate authentication to the request.
   Using cloud-specific protocols will at least give users an option to register a paid account and
   allow the data provider to properly charge for access.
   STAC-specific tools in turn can look for the cloud-specific protocols and know to use the requestor pays feature for that specific cloud platform.

## Consistent URIs

Links in STAC can be [absolute or relative](best-practices-catalog-and-collection.md#use-of-links).

Relative links must be resolved against a base URL, which is the absolute URI given in the link with the relation type `self`.
If a `self` link is not provided, the URL from which the document was retrieved is used as the base URL.
If the document was retrieved after a redirect, this is the URL that was redirected to
(see [RFC 3986, section 5.1.3](https://datatracker.ietf.org/doc/html/rfc3986#section-5.1.3)).
If neither of them is available, relative links can usually not be resolved and the behavior is undefined.

Many clients use the URL from which they retrieved the document as the base URL, even if a `self` link is provided.
This is the default in RFC 3986, which also applies to OGC API standards as they don't define a base URL.
Both approaches only lead to the same result if the `self` link is the URL under which the document is served.
Therefore, the `self` link should always contain the URL from which the document can be retrieved.
If the document has a preferred location that differs from this URL,
for example for a copy of a catalog on a mirror or for an API response that was generated from a static catalog,
provide the preferred location in a link with the relation type
[`canonical`](best-practices-catalog-and-collection.md#using-relation-types) instead of in the `self` link.

To resolve relative URIs, the base URIs must be precise and consistent.
Having or not having a trailing slash is significant (except if no path component is provided in a URL, see example 8).
Without a trailing slash, the last path component is identified as a "file" and will be removed while resolving URLs.
This means that if the trailing slash is missing for a folder,
a relative link would need to include the last path component again to resolve correctly (see example 4).
Relative URLs that start with a slash only depend on the scheme and host of the base URL,
not on its path (see examples 10 and 11).
They break if the catalog is moved to another path, for example behind a proxy.

In static catalogs, it is recommended to link to files (e.g., `catalog.json`) instead of folders,
and to consistently add a slash at the end of a URL if it points to a folder.

**Examples:**

| #  | Base URL                                  | Relative URL        | Resolved URL                                  |
| -- | ----------------------------------------- | ------------------- | --------------------------------------------- |
| 1  | `https://example.com/folder/catalog.json` | `item.json`         | `https://example.com/folder/item.json`        |
| 2  | `https://example.com/folder`              | `item.json`         | `https://example.com/item.json`               |
| 3  | `https://example.com/folder/`             | `item.json`         | `https://example.com/folder/item.json`        |
| 4  | `https://example.com/folder`              | `folder/item.json`  | `https://example.com/folder/item.json`        |
| 5  | `https://example.com/folder/`             | `folder/item.json`  | `https://example.com/folder/folder/item.json` |
| 6  | `https://example.com/another/folder`      | `../item.json`      | `https://example.com/item.json`               |
| 7  | `https://example.com/another/folder/`     | `../item.json`      | `https://example.com/another/item.json`       |
| 8  | `https://example.com`                     | `folder/item.json`  | `https://example.com/folder/item.json`        |
| 9  | `https://example.com/`                    | `folder/item.json`  | `https://example.com/folder/item.json`        |
| 10 | `https://example.com/another/folder`      | `/folder/item.json` | `https://example.com/folder/item.json`        |
| 11 | `https://example.com/another/folder/`     | `/folder/item.json` | `https://example.com/folder/item.json`        |

The relative URLs `folder/item.json` and `./folder/item.json` are equivalent.

### Documents available under multiple URLs

A document that a server returns both with and without a trailing slash,
for example at `https://example.com/stac/test` and at `https://example.com/stac/test/`,
cannot contain relative URLs of the usual form that resolve correctly for both requests
if clients use the URL from which the document was retrieved as the base URL:

| Relative URL       | Retrieved from `https://example.com/stac/test` | Retrieved from `https://example.com/stac/test/`    |
| ------------------ | ---------------------------------------------- | -------------------------------------------------- |
| `items`            | `https://example.com/stac/items` (wrong)       | `https://example.com/stac/test/items`              |
| `test/items`       | `https://example.com/stac/test/items`          | `https://example.com/stac/test/test/items` (wrong) |
| `/stac/test/items` | `https://example.com/stac/test/items`          | `https://example.com/stac/test/items`              |

This mostly affects APIs. STAC API and OGC API - Features define paths without a trailing slash
(e.g., `/collections/{collectionId}`), but many servers also accept the same path with a trailing slash.
To avoid issues in APIs, it is recommended to:

- use absolute URLs in all links (see [dynamic catalogs](best-practices-catalog-and-collection.md#should-you-use-relative-or-absolute-links)),
- if a server also accepts paths with a trailing slash,
  either redirect them to the path without the trailing slash
  (relative URLs are then resolved against the URL that was redirected to),
  or only use absolute URLs and relative URLs that start with a slash in the response, and
- provide a `self` link in each STAC entity, including the Items in an ItemCollection,
  so that clients never need to construct URLs themselves.
