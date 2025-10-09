# STAC Asset and Link Best Practices

## Table of Contents

- [Common Use Cases of Additional Fields for Assets](#common-use-cases-of-additional-fields-for-assets)
- [Working with Media Types](#working-with-media-types)
- [Asset Roles](#asset-roles)
- [Bands](#bands)

## Common Use Cases of Additional Fields for Assets

As [described in the Item spec](https://github.com/radiantearth/stac-spec/blob/master/commons/assets.md#additional-fields), it is possible to use fields typically
found in Item properties at the asset level. This mechanism of overriding or providing Item Properties only in the Assets 
makes discovery more difficult and should generally be avoided. However, there are some core and extension fields for which 
providing them at the Asset level can prove to be very useful for using the data.

- `datetime`: Provide individual timestamp on an Item, in case the Item has a `start_datetime` and `end_datetime`,
  but an Asset is for one specific time.
- `gsd` ([Common Metadata](https://github.com/radiantearth/stac-spec/blob/master/commons/common-metadata.md#instrument)): Specify some assets that represent instruments 
  with different spatial resolution than the overall best resolution. Note this should not be used for different 
  spatial resolutions due to specific processing of assets - look into the [raster 
  extension](https://github.com/stac-extensions/raster) for that use case.
- `bands` (e.g. in combination with the [EO extension](https://github.com/stac-extensions/eo/)):
  Provide spectral band information, and order of bands, within an individual asset.
- `proj:code`/`proj:wkt2`/`proj:projjson` ([projection extension](https://github.com/stac-extensions/projection/)):
  Specify different projection for some assets. If the projection is different
  for all assets it should probably not be provided as an Item property. If most assets are one projection, and there is 
  a single reprojected version (such as a Web Mercator preview image), it is sensible to specify the main projection in the 
  Item and the alternate projection for the affected asset(s).
- `proj:shape`/`proj:transform` ([projection extension](https://github.com/stac-extensions/projection/)):
  If assets have different spatial resolutions and slightly different exact bounding boxes,
  specify these per asset to indicate the size of the asset in pixels and its exact GeoTransform in the native projection.
- `sar:polarizations` ([sar extension](https://github.com/stac-extensions/sar)):
  Provide the polarization content and ordering of a specific asset.
- `sar:product_type` ([sar extension](https://github.com/stac-extensions/sar)):
  If mixing multiple product types within a single Item, this can be used to specify the product_type for each asset.

## Titles

It is recommended to always provide link titles.
The link titles should always reflect the title of the entity it refers to.
For example, if a STAC Item links to a STAC Collection, the value of the `title` property in the link with relation type `collection`
should **exactly** match the value of the `title` property in the STAC Collection.
Implementations should ensure that link titles are always synchronized so that inconsistencies don't occur.

Providing titles enables users to search and navigate more easily through STAC catalogs,
makes the links more predictable, and may prevent "flickering" in user-interfaces such as STAC Browser.

If the entity that a link refers to has no title, the value of the `id` can be considered as an alternative.

## Working with Media Types

[Media Types](https://en.wikipedia.org/wiki/Media_type) are a key element that enables STAC to be a rich source of information for
clients. The best practice is to use as specific of a media type as possible (so if a file is a GeoJSON then don't use a JSON
media type), and to use [registered](https://www.iana.org/assignments/media-types/media-types.xhtml) IANA types as much as possible.

For hierarchical links (e.g. relation types `root`, `parent`, `child`, `item`) it is important that
clients filter for the corresponding STAC media types
(e.g. `application/json` for all relation types and/or `application/geo+json` for relation type `item`). 
Hierarchical links with other media types (e.g. `text/html`) may be present for hierarchical links,
especially in STAC implementations that are also implementing OGC API - Records.

### Common Media Types in STAC

The following table lists a number of commonly used media types in STAC. The first two (GeoTIFF and COG) are not fully standardized 
yet, but reflect the community consensus direction. There are many IANA registered types that commonly show up in STAC. The 
following table lists some of the most common ones you may encounter or use.

| Media Type                                                 | Description                                                                                                                                                                                                                               |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image/tiff; application=geotiff`                          | GeoTIFF with standardized georeferencing metadata                                                                                                                                                                                         |
| `image/tiff; application=geotiff; profile=cloud-optimized` | [Cloud Optimized GeoTIFF](https://www.cogeo.org/) (unofficial). Once there is an [official media type](http://osgeo-org.1560.x6.nabble.com/Media-type-tc5411498.html) it will be added and the custom media type here will be deprecated. |
| `image/jp2`                                                | JPEG 2000                                                                                                                                                                                                                                 |
| `image/png`                                                | Visual PNGs (e.g. thumbnails)                                                                                                                                                                                                             |
| `image/jpeg`                                               | Visual JPEGs (e.g. thumbnails, oblique)                                                                                                                                                                                                   |
| `text/xml` or `application/xml`                            | XML metadata [RFC 7303](https://www.ietf.org/rfc/rfc7303.txt)                                                                                                                                                                             |
| `application/json`                                         | A JSON file (often metadata, or [labels](https://github.com/radiantearth/stac-spec/tree/master/extensions/label#labels-required))                                                                                                         |
| `text/plain`                                               | Plain text (often metadata)                                                                                                                                                                                                               |
| `application/geo+json`                                     | [GeoJSON](https://geojson.org/)                                                                                                                                                                                                           |
| `application/geopackage+sqlite3`                           | [GeoPackage](https://www.geopackage.org/)                                                                                                                                                                                                 |
| `application/x-hdf5`                                       | Hierarchical Data Format version 5                                                                                                                                                                                                        |
| `application/x-hdf`                                        | Hierarchical Data Format versions 4 and earlier.                                                                                                                                                                                          |
| `application/vnd.laszip+copc`                              | [COPC](https://copc.io/) Cloud optimized PointCloud                                                                                                                                                                                       |
| `application/vnd.apache.parquet`                           | Apache [Geoparquet](https://geoparquet.org/)                                                                                                                                                                                              |
| `application/3dtiles+json`                                 | [OGC 3D Tiles](https://www.ogc.org/standard/3dtiles/)                                                                                                                                                                                     |
| `application/vnd.pmtiles`                                  | Protomaps [PMTiles](https://github.com/protomaps/PMTiles/blob/main/spec/v3/spec.md)                                                                                                                                                       |

*Deprecation notice: GeoTiff previously used the media type `image/vnd.stac.geotiff` and
Cloud Optimized GeoTiffs used `image/vnd.stac.geotiff; profile=cloud-optimized`.
Both can still appear in old STAC implementations, but are deprecated and should be replaced.

#### Formats with no registered media type

This section gives recommendations on what to do if you have a format in your links or assets
that does not have an IANA registered type.
Ideally every media type used is on the [IANA registry](https://www.iana.org/assignments/media-types/media-types.xhtml). If
you are using a format that is not on that list we recommend you use
[custom content type](https://restcookbook.com/Resources/using-custom-content-types/).
These typically use the `vnd.` prefix, see [RFC 6838 section-3.2](https://tools.ietf.org/html/rfc6838#section-3.2).
Ideally the format provider will actually
register the media type with IANA, so that other STAC clients can find it easily. But if you are only using it internally it is 
[acceptable to not register](https://stackoverflow.com/questions/29121241/custom-content-type-is-registering-with-iana-mandatory) 
it. It is relatively easy to [register](https://www.iana.org/form/media-types) a `vnd` media type.

## Asset Roles

[Asset roles](https://github.com/radiantearth/stac-spec/blob/master/commons/assets.md#roles) are used to describe what each asset is used for. They are particular useful 
when several assets have the same media type, such as when an Item has a multispectral analytic asset, a 3-band full resolution 
visual asset, a down-sampled preview asset, and a cloud mask asset, all stored as Cloud Optimized GeoTIFF (COG) images. It is 
recommended to use at least one role for every asset available, and using multiple roles often makes sense. For example you'd use
both `data` and `reflectance` if your main data asset is processed to reflectance, or `metadata` and `cloud` for an asset that 
is a cloud mask, since a mask is considered a form of metadata (it's information about the data). Or if a single asset represents
several types of 'unusable data' it might include `metadata`, `cloud`, `cloud-shadow` and `snow-ice`. If there is not a clear
role then just pick a sensible name for the role. You are encouraged to add it to the list below and/or
in an extension if you think the new role will have broader applicability. 

### List of Asset Roles

There are a number of roles that are commonly used in practice, which we recommend to reuse as much as possible.
If you can't find suitable roles, feel free to suggest more.

| Role Name  | Description                                                                                                                                                                                                                                                                                     |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| data       | The data itself, excluding the metadata.                                                                                                                                                                                                                                                        |
| metadata   | Metadata sidecar files describing the data, for example a Landsat-8 MTL file.                                                                                                                                                                                                                   |
| thumbnail  | An asset that represents a thumbnail of the Item or Collection, typically a RGB or grayscale image primarily for human consumption, low resolution, restricted spatial extent, and displayable in a web browser without scripts or extensions.                                                  |
| overview   | An asset that represents a more detailed overview of the Item or Collection, typically a RGB or grayscale image primarily for human consumption, medium resolution, full spatial extent, in a file format that's can be visualized easily (e.g., Cloud-Optimized GeoTiff).                      |
| visual     | An asset that represents a detailed overview of the Item or Collection, typically a RGB or grayscale image primarily for human consumption, high or native resolution (often sharpened), full spatial extent, in a file format that's can be visualized easily (e.g., Cloud-Optimized GeoTiff). |
| date       | An asset that provides per-pixel acquisition timestamps, typically serving as metadata to another asset                                                                                                                                                                                         |
| graphic    | Supporting plot, illustration, or graph associated with the Item                                                                                                                                                                                                                                |
| data-mask  | File indicating if corresponding pixels have Valid data and various types of invalid data                                                                                                                                                                                                       |
| snow-ice   | Points to a file that indicates whether a pixel is assessed as being snow/ice or not.                                                                                                                                                                                                           |
| land-water | Points to a file that indicates whether a pixel is assessed as being land or water.                                                                                                                                                                                                             |
| water-mask | Points to a file that indicates whether a pixel is assessed as being water (e.g. flooding map).                                                                                                                                                                                                 |
| iso-19115  | Points to an [ISO 19115](https://www.iso.org/standard/53798.html) metadata file                                                                                                                                                                                                                 |

Additional roles are defined in the various extensions, for example:

- [EO Extension](https://github.com/stac-extensions/eo/blob/main/README.md#best-practices)
- [View Extension](https://github.com/stac-extensions/view/blob/main/README.md#best-practices)
- [SAR Extension](https://github.com/stac-extensions/sar/blob/main/README.md#best-practices)

The roles `thumbnail`, `overview` and `visual` are very similar. To make choosing the right role easier, please consult the table below.

They should usually be a RGB or grayscale image, which are primarily intended for human consumption, e.g., through a web browser.
It can complement assets where one band is per file (like Landsat), by providing the key display bands combined,
or can complement assets where many non-visible bands are included, by being a lighter weight file that just has the bands needed for display.

Roles should also be combined, e.g., `thumbnail` and `overview` if the recommendations are all met.

If your data for the Item does not come with a thumbnail already we do recommend generating one, which can be done quite easily with [GDAL](https://gdal.org/) or [Rasterio](https://rasterio.readthedocs.io/en/latest/).

| Role                           | thumbnail                                           | overview                                                           | visual                                                                                       |
| ------------------------------ | --------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Resolution                     | Low                                                 | Medium                                                             | Native / High                                                                                |
| *Recommended* Image dimensions | < 600x600 px                                        | < 5000x500 px                                                      | any size                                                                                     |
| *Recommended* File Formats     | PNG, JPEG, GIF, WebP                                | PNG, JPEG, WebP, COG                                               | COG, other cloud-native and/or tiled file formats with pyramids, ...                         |
| Spatial extent                 | Limited                                             | Full                                                               | Full                                                                                         |
| Use case                       | Quick overview, often in lists of items/collections | Display for a single Item/Collection, sometimes shown on a web map | Display for a single Item/Collection, often shown on a map, may be displayed in GIS software |

The files offered for the roles `thumbnail`, `overview` and `visual` should be accessible via HTTP(S).
If the [Alternate Asset Extension](https://github.com/stac-extensions/alternate-assets) is used,
the default access mechanism should be HTTP(S).

## Bands

As of STAC 1.1, the `bands` array can be used in combination with property inheritance to provide users with more flexibility.
The following best practices should be considered, especially when migrating from `eo:bands` and `raster:bands`.

### Single band

Single band assets can be defined in two ways.
Properties can be defined in the `bands` array or in the assets directly.

Example using the `bands` array:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "bands": [
        {
          "data_type": "uint16",
          "eo:common_name": "red",
          "raster:spatial_resolution": 10
        }
      ]
    }
  }
}
```

Example without bands:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "data_type": "uint16",
      "eo:common_name": "red",
      "raster:spatial_resolution": 10
    }
  }
}
```

STAC recommends that single band assets should only use the `bands` array in the following cases:

1. **It's important in to convey that a band is present in the asset.**
   - This is the case if the data access mechanism requires you to specify the name of index of the band to retrieve the data,
     then the band should be specified as such.
   - This is also the case if the band has a specific name.
     The `name` property is only available under the `bands` array and as such can't be specified for the Asset.
2. **It is important that the (often spectral) band is part of a set of bands.**
   - For example, if the `bands` array is exposed in the Collection Summaries,
     then a `bands` array should be defined in the Items or Item Assets as otherwise there's nothing to summarize.
3. **Individual bands are repeated in different assets.**
   - This may happen if you provide assets with different resolutions or file formats.
     The `name` property with the same value should be used so that users can identify that the bands are the same.
     It also enables clients to easily combine and summarize the bands.

### Multiple bands

Generally, all properties that have the same value across all bands should not be listed in the `bands` array but in the assets directly.

For example, if your `bands` array in an asset is defined as follows:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "bands": [
        {
          "name": "r",
          "eo:common_name": "red",
          "data_type": "uint16",
          "raster:spatial_resolution": 10
        },
        {
          "name": "g",
          "eo:common_name": "green",
          "data_type": "uint16",
          "raster:spatial_resolution": 10
        },
        {
          "name": "b",
          "eo:common_name": "blue",
          "data_type": "uint16",
          "raster:spatial_resolution": 10
        }
      ]
    }
  }
}
```

The `data_type` and `raster:spatial_resolution` has the same value for all bands.
As such you can deduplicate those properties and list them in the asset directly:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "data_type": "uint16",
      "raster:spatial_resolution": 10,
      "bands": [
        {
          "name": "r",
          "eo:common_name": "red"
        },
        {
          "name": "g",
          "eo:common_name": "green"
        },
        {
          "name": "b",
          "eo:common_name": "blue"
        }
      ]
    }
  }
}
```

### Band migration

It should be relatively simple to migrate from STAC 1.0 (i.e. `eo:bands` and/or `raster:bands`) to the new `bands` array.

Usually, you can simply merge each object on a by-index basis.
Nevertheless, you should consider deduplicating properties with the same values across all bands to the Asset.
For some fields, you need to add the extension prefix of the `eo` or `raster` extension to the property name.

STAC 1.0 example:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "eo:bands": [
        {
          "name": "r",
          "common_name": "red"
        },
        {
          "name": "g",
          "common_name": "green"
        },
        {
          "name": "b",
          "common_name": "blue"
        },
        {
          "name": "nir",
          "common_name": "nir"
        }
      ],
      "raster:bands": [
        {
          "data_type": "uint16",
          "spatial_resolution": 10,
          "sampling": "area"
        },
        {
          "data_type": "uint16",
          "spatial_resolution": 10,
          "sampling": "area"
        },
        {
          "data_type": "uint16",
          "spatial_resolution": 10,
          "sampling": "area"
        },
        {
          "data_type": "uint16",
          "spatial_resolution": 30,
          "sampling": "area"
        }
      ]
    }
  }
}
```

After migrating to STAC 1.1 this is ideally provided as follows:

```json
{
  "assets": {
    "example": {
      "href": "example.tif",
      "data_type": "uint16",
      "raster:sampling": "area",
      "raster:spatial_resolution": 10,
      "bands": [
        {
          "name": "r",
          "eo:common_name": "red"
        },
        {
          "name": "g",
          "eo:common_name": "green"
        },
        {
          "name": "b",
          "eo:common_name": "blue"
        },
        {
          "name": "nir",
          "eo:common_name": "nir",
          "raster:spatial_resolution": 30
        }
      ]
    }
  }
}
```

The following was done:

- The arrays have been merged into a single property `bands`.
- The properties `common_name` and `spatial_resolution` were renamed to include the extension prefixes.
- The properties `data_type` and `raster:sampling` (renamed from `sampling`) were deduplicated to the Asset
  as the values were the same across all bands.
- The `spatial_resolution` was also deduplicated, i.e. `10` is provided on the asset level,
  which is inherited by the bands unless explicitly overridden.
  Therefore, the `nir` band overrides the value `10` with a value of `30`.

As a result, the new `bands` array is more lightweight and easier to handle.
