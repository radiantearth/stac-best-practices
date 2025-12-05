# STAC API Best Practices

_This page is very much a work in progress_

## Table of Contents

- [Static to Dynamic best practices](#static-to-dynamic-best-practices)
  - [Ingestion and links](#ingestion-and-links)
  - [Keep catalogs in sync with cloud notification and queue services](#keep-catalogs-in-sync-with-cloud-notification-and-queue-services)


## Static to Dynamic best practices

Many implementors are using static catalogs to be the reliable core of their dynamic services, or layering their STAC API
on top of any static catalog that is published. These are some recommendations on how to handle this:

### Ingestion and links

Implementors have found that it's best to 'ingest' a static STAC into an internal datastore (often elasticsearch, but a 
traditional database could work fine too) and then generate the full STAC API responses from that internal representation.
There are instances that have the API refer directly to the static STAC Items, but this only works well if the static STAC 
catalog is an 'absolute published catalog'. So the recommendation is to always use absolute links - either in the static 
published catalog, or to create new absolute links for the STAC search/ endpoint 
responses, with the API's location at the base url. The `/` endpoint with the catalog could either link directly
to the static catalog, or can follow the 'dynamic catalog layout' recommendations above with a new set of URL's.

Ideally each Item would use its `links` to provide a reference back to the static location. The location of the static
Item should be treated as the canonical location, as the generated API is more likely to move or be temporarily down. The
spec provides the `derived_from` rel field, which fits well enough, but `canonical` is likely the more appropriate one
as everything but the links should be the same.

### Keep catalogs in sync with cloud notification and queue services

There is a set of emerging practices to use services like Amazon's Simple Queue Service (SQS)
and Simple Notification Service (SNS) to keep catalogs in sync.
There is a great [blog post](https://aws.amazon.com/blogs/publicsector/keeping-a-spatiotemporal-asset-catalog-stac-up-to-date-with-sns-sqs/)
on the CBERS STAC implementation on AWS.
The core idea is that a static catalog should emit a notification whenever it changes. The recommendation for SNS is to use the STAC 
Item JSON as the message body, with some fields such as a scene’s datetime and geographic bounding box that allows 
basic geographic filtering from listeners. 

The dynamic STAC API would then listen to the notifications and update its internal datastore whenever new data comes into
the static catalog. Implementors have had success using AWS Lambda to do a full 'serverless' updating of the elasticsearch
database, but it could just as easily be a server-based process.
