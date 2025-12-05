# STAC Item Best Practices

## Table of Contents

- [Field and ID formatting](#item-ids)
- [Searchable Identifiers](#searchable-identifiers)
- [Field selection and Metadata Linking](#field-selection-and-metadata-linking)
- [Datetime selection](#datetime-selection)
- [Unlocated Items](#unlocated-items)
- [Representing Vector Layers in STAC](#representing-vector-layers-in-stac)

## Item IDs

When defining one's STAC properties and fields there are many choices to make on how to name various aspects of one's
data. One of the key properties is the ID. The specification is quite flexible on ID's, primarily so that existing
providers can easily use their same ID when they translate their data into STAC.
It is STRONGLY RECOMMENDED that an item ID is unique per collection.
The use of URI or file path reserved characters such as `:` or `/` is discouraged since this will
result in [percented encoded](https://tools.ietf.org/html/rfc3986#section-2) [STAC API](https://github.com/radiantearth/stac-api-spec)
endpoints and it prevents the use of IDs as file names as recommended in the [catalog layout](best-practices-catalog-and-collection.md#catalog-layout) best practices.

## Searchable Identifiers

When coming up with values for fields that contain searchable identifiers of some sort, like `constellation` or `platform`,
it is recommended that the identifiers consist of only lowercase characters, numbers, `_`, and `-`.
Examples include `sentinel-1a` (Sentinel-1), `landsat-8` (Landsat-8) and `envisat` (Envisat).
This is to provide consistency for search across Collections, so that people can just search for `landsat-8`,
instead of thinking through all the ways providers might have chosen to name it.

## Field selection and Metadata Linking

In general STAC aims to be oriented around **search**, centered on the core fields that users will want to search on to find
data. The core is space and time, but there are often other metadata fields that are useful. While the specification is
flexible enough that providers can fill it with tens or even hundreds of fields of metadata, that is not recommended. When
adding fields ask:

- Does this field help with search and discovery of data?
- Does this field tell me how to access the data?

If the purpose of the field is solely to provide information that is not contained within the data, consider putting
that metadata in external metadata and refer to it via an
[Asset Object](https://github.com/radiantearth/stac-spec/blob/master/commons/assets.md#asset-object).
There is a lot of metadata that is only of relevance
to loading and processing data, and while STAC does not prohibit providers from putting those type of fields in their items,
it is not recommended.

For more information on what fields to include and where to include them in the STAC hierarchy, read the
[Metadata and Extension Best Practices](best-practices-metadata-and-extensions.md).

## Datetime selection

The `datetime` field in a STAC Item's properties is one of the most important parts of a STAC Item, providing the T (temporal) of
STAC. And it can also be one of the most confusing, especially for data that covers a range of times. For many types of data it
is straightforward - it is the capture or acquisition time. But often data is processed from a range of captures - drones usually
gather a set of images over an hour and put them into a single image, mosaics combine data from several months, and data cubes
represent slices of data over a range of time. For all these cases the recommended path is to use `start_datetime` and
`end_datetime` fields from [common metadata](https://github.com/radiantearth/stac-spec/blob/master/commons/common-metadata.md#date-and-time-range). The specification does allow one to set the
`datetime` field to `null`, but it is strongly recommended to populate the single `datetime` field, as that is what many clients
will search on. If it is at all possible to pick a nominal or representative datetime then that should be used. But sometimes that
is not possible, like a data cube that covers a time range from 1900 to 2000. Setting the datetime as 1950 would lead to it not
being found if you searched 1990 to 2000.

Extensions that describe particular types of data can and should define their `datetime` field to be more specific. For example
a MODIS 8 day composite image can define the `datetime` to be the nominal date halfway between the two ranges. Another data type
might choose to have `datetime` be the start. The key is to put in a date and time that will be useful for search, as that is
the focus of STAC. If `datetime` is set to `null` then it is strongly recommended to use it in conjunction with an extension
that explains why it should not be set for that type of data.

## Unlocated Items

Though the [GeoJSON standard](https://tools.ietf.org/html/rfc7946) allows null geometries, in STAC we strongly recommend
that every item have a geometry, since the general expectation of someone using a SpatioTemporal Catalog is to be able to query
all data by space and time. But there are some use cases where it can make sense to create a STAC Item before it gets
a geometry. The most common of these is 'level 1' satellite data, where an image is downlinked and cataloged before it has
been geospatially located.

The recommendation for data that does not yet have a location is to follow the GeoJSON concept that it is an ['unlocated'
feature](https://tools.ietf.org/html/rfc7946#section-3.2). So if the catalog has data that is not located then it can follow
GeoJSON and set the geometry to null. Though normally required, in this case the `bbox` field should not be included.

Note that this recommendation is only for cases where data does not yet have a geometry and it cannot be estimated. There
are further details on the two most commonly requested desired use cases for setting geometry to null:

### Unrectified Satellite Data

Most satellite data is downlinked without information that precisely describes where it is located on Earth. A satellite
imagery processing pipeline will always attempt to locate it, but often that process takes a number of hours, or never
quite completes (like when it is too cloudy). It can be useful to start to populate the Item before it has a geometry.
In this case the recommendation is to use the 'estimated' position from the satellite, to populate at least the bounding box,
and use the same broad bounds for the geometry (or leaving it null) until there is precise ground lock. This estimation is
usually done by onboard equipment, like GPS or star trackers, but can be off by kilometers or more. But it is very useful for
STAC users to be able to at least find approximate area in their searches. A commonly used field for communicating ground lock
is not yet established, but likely should be (an extension proposal would be appreciated).  If there is no way to provide an
estimate then the data can be assigned a null geometry and no `bbox`, as described above. But the data will likely not
show up in STAC API searches, as most will at least implicitly use a geometry. Though this section is written with
satellite data in mind, one can easily imagine other data types that start with a less precise geometry but have it
refined after processing.

### Data that is not spatial

The other case that often comes up is people who love STAC and want to use it to catalog everything they have, even if it is
not spatial. This use case is not currently supported by STAC, as we are focused on data that is both temporal and spatial
in nature. The [OGC API - Records](https://github.com/opengeospatial/ogcapi-records) is an emerging standard that likely
will be able to handle a wider range of data than STAC. It builds on [OGC API -
Features](https://github.com/opengeospatial/ogcapi-features) just like [STAC API](https://github.com/radiantearth/stac-api-spec/)
does. Using [Collection Assets](https://github.com/radiantearth/stac-spec/blob/master/collection-spec/collection-spec.md#assets) may also provide an option for some
use cases.

## Representing Vector Layers in STAC

Many implementors are tempted to try to use STAC for 'everything', using it as a universal catalog of all their 'stuff'.
The main route considered is to use STAC to describe vector layers, putting a shapefile or [geopackage](http://geopackage.org)
as the `asset`. Though there is nothing in the specification that *prevents* this, it is not really the right level of
abstraction. A shapefile or geopackage corresponds to a Collection, not a single Item. The ideal thing to do with
one of those is to serve it with [OGC API - Features](https://github.com/opengeospatial/ogcapi-features) standard. This
allows each feature in the shapefile/geopackage to be represented online, and enables querying of the actual data. If
that is not possible then the appropriate way to handle Collection-level search is with the
[OGC API - Records](https://github.com/opengeospatial/ogcapi-records) standard, which is a 'brother' specification of STAC API.
Both are compliant with OGC API - Features, adding richer search capabilities to enable finding of data.
