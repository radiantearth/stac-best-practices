# Metadata and Extension Best Practices <!-- omit in toc -->

- [Introduction](#introduction)
- [General Data Access](#general-data-access)
- [Spectral Measurements](#spectral-measurements)
- [SAR Measurements](#sar-measurements)
- [Datacubes](#datacubes)
- [Other Measurements](#other-measurements)
- [Collections](#collections)
- [Credits](#credits)

## Introduction

The following specifications and extensions are discussed in this document:

- [STAC v1.1](https://github.com/radiantearth/stac-spec/tree/v1.1.0) with [common metadata](https://github.com/radiantearth/stac-spec/blob/v1.1.0/commons/common-metadata.md)
- [Authentication Extension v1.1 (or later 1.x)](https://github.com/stac-extensions/authentication)
- [Classification Extension v2.x](https://github.com/stac-extensions/classification)
- [Datacube Extension v2.x](https://github.com/stac-extensions/datacube)
- [Electro Optical (EO) Extension v2.x](https://github.com/stac-extensions/eo)
- [Product Extension v1.x](https://github.com/stac-extensions/product)
- [Projection Extension v2.x](https://github.com/stac-extensions/projection)
- [Raster Extension v2.x](https://github.com/stac-extensions/raster)
- [SAR Extension v1.2 (or later 1.x)](https://github.com/stac-extensions/sar)
- [Storage Extension v2.x](https://github.com/stac-extensions/storage)

> [!NOTE]  
> This document is written based on the version numbers specified in the list above.
> The recommendations can mostly be backported to other versions of STAC and other versions of extensions (e.g. the use of `proj:epsg` instead of `proj:code`).
> We recommend to implement the versions below though.


All Fields should be provided in the scope that's required for the data without duplicating information.
For example, if the data type differs per band, you'd want to provide the data type per band.
If the data type is the same across all bands, provide the data type per asset.
If the data type is the same across all assets, provide the data type per item.

Similarly provide the best gsd at the item level, and if any assets have a different spatial
resolution provide the gsd within those particular assets to override the value inherited from the item.

The same principles apply for collection-level assets. As much as possible metadata should be provided at
the collection-level

## General Data Access

Two behavioral extensions are recommended that don't describe the data itself but the data access:

- **Authentication Extension** (v1.1 or a later non-breaking version)

  - Throughout the whole implementation, all entries in `links` and `assets` should provide a `auth:refs` whenever authentication is required.
    `auth:refs` refers to `auth:schemes`, which has to be provided in the same file.

    Note that many authentication schemes still need additional information for authentication, e.g. OpenID Connect and OAuth need a client (client ID and secret).

- **Storage Extension** (v2.x)

  - Throughout the whole implementation, all entries in `assets` should provide a `storage:refs` whenever a cloud storage needs to be accessed that needs a different access mechanism than HTTP.
    `storage:refs` refers to `storage:schemes`, which has to be provided in the same file.

    Note that the Storage Extension is still evolving and has a limited set of cloud stores defined.
    Please open a PR if your cloud store is not available yet.

## General fields

- **Common metadata**
  - `datetime` Provide individual timestamp on an Item, in case the Item has a `start_datetime` and `end_datetime`,
  but an Asset is for one specific time.
  - `gsd` ([Common Metadata](commons/common-metadata.md#instrument)): Specify overall best resolution at the item level. Specify at the
  asset level to represent instruments with different spatial resolutions. Note this should not be used for different 
  spatial resolutions due to specific processing of assets - look into the [raster 
  extension](https://github.com/stac-extensions/raster) for that use case.
  - `data_type` 
  - `nodata` (if applicable)
  - `unit` (if applicable)
  
- **Projection Extension** (v2.x)
  
  - `proj:code`, or alternatively `proj:wkt2` or `proj:projjson` if no code is available for the CRS.
  
    `proj:code` can be set to `null` if one of the other two fields is provided.
    `proj:wkt2` and `proj:projjson` should never be set to `null`.

    Typically specified at the Item level. If the projection is different
    for all assets it should not be provided as an Item property. If most assets are one projection, and there is 
    a single reprojected version (such as a Web Mercator preview image), it is sensible to specify the main projection in the 
    Item and the alternate projection for the affected asset(s).

  - Two of the following: `proj:shape`, `proj:transform`, `proj:bbox`.
  
    `proj:bbox` should be omitted if the values match `bbox` property exactly (i.e. if `proj:code` "is a geographic coordinate reference system, using the World Geodetic System 1984 (WGS 84) datum, with longitude and latitude units of decimal degrees" (see [RFC 7946, ch. 4](https://datatracker.ietf.org/doc/html/rfc7946#section-4))).

    Typically specified at the Item level. If assets have different spatial resolutions and slightly different exact bounding boxes,
    specify these per asset to indicate the size of the asset in pixels and its exact GeoTransform in the native projection.
  
- **Raster Extension** (v2.x)

  - `raster:spatial_resolution` (if multiple resolutions are provided in the Asset, otherwise in the Item)
  - `raster:scale`
  - `raster:offset`

  **CAUTION:** The scale and offset should only be provided if the processing software explicitly has to apply the scale and the offset, e.g. for Sentinel-2 digital numbers.
  For netCDF files - where the scale and offset is applied automatically according to the netCDF specification - the scale and offset should **not** be provided.

- **Classification Extension** (v2.x)

  The Classification Extension should only be provided for **categorical data**.
  
  - `classification:classes`
  - `classification:bitfields` (if applicable)

## Spectral Measurements

- **Common metadata**

  - `bands` (in the order they should appear in concatenated data structure)

- **Electro Optical (EO) Extension** (v2.x)
  
  - `name`
  
    For users of the datacube extension, the names should also be provided as the `values` for the band dimension.
  - `eo:common_name` (if applicable)
  - `eo:center_wavelength` and `eo:full_width_half_max`

## SAR Measurements

- **[SAR Extension](https://github.com/stac-extensions/sar)**  (v1.2 or a later non-breaking version)
  - `sar:polarizations` (in the order they should appear in concatenated data structure)

- **[Product Extension](https://github.com/stac-extensions/product)**
  - `product:type`
   
    If mixing multiple product types within a single Item, this can be used to specify per asset.

## Datacubes

Items should contain as much information as possible for the construction of datacubes without
accessing data referenced by the assets. As such the following information should be provided:

- Exact **spatial and temporal extents** (if not provided through the datacube extension)
- **Datacube extension** The datacube extension can be used to indicate how to construct the datacube even for non-datacube data. 
  *Note: Usually only extents can be provided but no specific dimension labels.*
- **Bands** (`bands`, providing the union of all available bands)
- **Item Assets** (`item_assets`)

If an asset refers to a datacube file format (e.g. netCDF, ZARR, GRIB), the following is recommended:

- **Datacube Extension** (v2.x)
  
  - For a single variable: `cube:dimensions` only
  - For multiple variables: `cube:variables` and `cube:dimensions`

> [!CAUTION]
> For Collections containing datacubes, fields (including `bands` or datacube fields) should be provided on the top-level 
and **not** in summaries. Summaries is reserved for properties summarizing the Item Properties, 


### Other Measurements

> [!NOTE]
> This will be detailed once use cases come up. We welcome contributions to this section.
>
> Generally speaking, users should follow any STAC best practices that exist for their types of measurements.
> The bands concept of STAC could be reused as well.

## Credits

This document is based on the [Open Data Cube](https://odc-stac.readthedocs.io/en/latest/stac-best-practice.html) and [GDAL inspired best practices in the projection extension](https://github.com/stac-extensions/projection?tab=readme-ov-file#best-practices).
