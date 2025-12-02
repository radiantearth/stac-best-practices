# STAC Zarr Best Practices

## Table of Contents

- [Introduction](#introduction)
- [Glossary](#glossary)
- [General Rules](#general-rules)
  - [STAC Object Type](#stac-object-type)
  - [Asset Organization](#asset-organization)
  - [Bands Representation](#bands-representation)
  - [Link Relationships and Types](#link-relationships-and-types)
  - [Link Templates](#link-templates)
- [STAC Extensions Requirements](#stac-extensions-requirements)
- [Use Cases](#use-cases)
  - [Scene-Based Zarr Stores (EOPF Products)](#scene-based-zarr-stores-eopf-products)
    - [Resolution based group representation](#resolution-based-group-representation)
    - [Multiscale reflectance group representation](#multiscale-reflectance-group-representation)
  - [CF-Compliant Climate and Weather Data](#cf-compliant-climate-and-weather-data)
  - [Virtual Zarr Stores](#virtual-zarr-stores)

## Introduction

This document provides best practices for representing Zarr stores (including native Zarr and virtual Zarr stores) in STAC. These guidelines address:

- STAC object types suitable for Zarr stores
- Asset hierarchy and organization for Zarr groups and arrays
- Discovery patterns for variables within multidimensional stores
- Integration with datacube extension for variable and dimension metadata
- Use of link relationships to reference Zarr stores
- Asset metadata for multi-resolution and hierarchical data structures

The following specifications and extensions are referenced in this document:

- [STAC v1.1](https://github.com/radiantearth/stac-spec/tree/v1.1.0)
- [Datacube Extension v2.x](https://github.com/stac-extensions/datacube)
- [Raster Extension v2.x](https://github.com/stac-extensions/raster)
- [CF Extension v1.x](https://github.com/stac-extensions/cf)
- [Electro Optical (EO) Extension v2.x](https://github.com/stac-extensions/eo)
- [Projection Extension v2.x](https://github.com/stac-extensions/projection)

## Glossary

### Zarr Terminology

See the [Zarr specification](https://zarr-specs.readthedocs.io/en/latest/v3/core/#concepts-and-terminology) for complete terminology.

Key terms:

- **Zarr Store**: A storage system that holds a Zarr hierarchy (e.g., a directory, zip file, or object store prefix) conforming to the [Zarr abstract store interface](https://zarr-specs.readthedocs.io/en/latest/v3/core/index.html#abstract-store-interface)
- **Group**: Any node - including the root node - in a Zarr hierarchy that may have child nodes.
- **Array**: A leaf node in a Zarr hierarchy that represents a multidimensional array with chunked storage
- **Chunk**: A contiguous region of an array that is stored within a single object

Conventions terms:

- **Multiscales Group**: A [convention](https://github.com/zarr-conventions/multiscales) for organizing multi-resolution data in Zarr stores, typically using a group of groups structure where each child group represents a different resolution.

### Xarray/NetCDF/CF-Specific Concepts

- **Variable**: A named array
- **Dimension**: The name of an axis of a multidimensional array (e.g., x, y, time, band)
- **Coordinate Variable**: A named array that provides coordinate values along a dimension
- **Data Variable**: A named array containing measured or derived data values

## General Rules

### STAC Object Type

Zarr groups and arrays can be represented by 2 STAC object types depending on the use case.

#### STAC Item

A STAC Item is appropriate when the Zarr group represents a particular location in space and/or time, such as:

- A satellite scene with multiple bands and resolutions
- A time-specific climate model run output with multiple variables and dimensions
- A single time slice of a time series
- A single multiscale dataset

Each STAC Item in the collection references its own separate Zarr store. This pattern is appropriate when individual scenes or time slices are stored independently, allowing for more flexible data management and access patterns. Each Item's assets reference its unique Zarr store via the [store link](#store-link-relationship).

#### STAC Collection

A STAC Collection is appropriate when the Zarr group represents multiple locations in space and/or time, such as:

- A time series of satellite scenes
- A climate model output spanning multiple time steps
- A set of multiscale datasets for different regions or time periods
- A collection of Zarr stores representing different variables or measurements

The entire collection is contained within a single Zarr store. In this case, the data are often organized using additional dimensions (e.g., time) within arrays to organize the data. Assets in the STAC Collection can reference specific groups within the store and use the datacube extension to describe the multidimensional structure.

### Asset Organization

1. **A Zarr asset href SHALL reference a group in the Zarr hierarchy**

   A single Zarr asset corresponds to a Zarr group, which may contain multiple arrays and subgroups.
   This is equivalent to an xarray Dataset or an xarray DataTree.

2. **Assets representing Zarr stores SHALL use the appropriate media type with version information**

   - Zarr v2: `"application/vnd.zarr; version=2"`
   - Zarr v3: `"application/vnd.zarr; version=3"`

   We also recommend using a `profile` parameter to indicate specific data models such as multiscales (see #metadata-requirements).

>[!NOTE]
> The proposed `profile` parameter is not an official part of the Zarr media type specification but is suggested here to enhance clarity regarding the data model used within the Zarr store.

3. **Bands within an asset SHALL reference available bands in a multispectral/multi-channel Zarr group**

   Individual variable arrays within the store SHOULD NOT be represented as separate assets.

   The appropriate level of detail to include within the bands object depends on how users will access the data. Section #bands-representation below provide details on how to represent bands within a Zarr asset.

#### Bands Representation

For many applications, it is useful to provide band-level metadata for spectral or multi-channel data stored in Zarr groups. This allows clients to discover available bands and their properties without needing to scan the Zarr hierarchy.

According to the organization of the data in the asset group and child arrays, bands can be represented in different ways described below.

A general rule is that every band SHALL correspond to an entry in the `bands` array of the hosting asset.
The best practices for `bands` from the [core specification](https://github.com/radiantearth/stac-spec/blob/master/best-practices.md#bands) also apply here.

##### Unique band per data variable

The most straightforward case is when each data variable corresponds to a unique band.

- The **Asset href SHALL point to the group** containing the data variables
- Each band in the `bands` array SHALL reference a data variable by using the `name` field

 ```json
"assets": {
  "reflectance": {
    // href pointing to a group within a Zarr store
    "href": "s3://bucket/path/data.zarr/measurements/reflectance/r10m",
    "type": "application/vnd+zarr; version=3",
    "bands": [
      // band object for the `b02` variable within the `reflectance/r10m` Zarr group
      {"name": "b02", "eo:common_name": "blue"},
      {"name": "b03", "eo:common_name": "green"},
      {"name": "b04", "eo:common_name": "red"},
    ]
  }
}
```

##### Multiple bands per data variable (band as dimension)

In some cases, a single data variable may contain multiple bands along a specific dimension (e.g., spectral bands stored as the dimension of the array). In this case, the band objects SHALL reference indexes within a variable, with the `name` field indicating both the variable name and the dimension index or label.

```json
"assets": {
  "reflectance": {
    "href": "s3://bucket/path/data.zarr/measurements",
    "type": "application/vnd+zarr; version=3",
    "bands": [
    {"name": "reflectance[band=blue]", "eo:common_name": "blue"},
    {"name": "reflectance[band=green]", "eo:common_name": "green"},
    {"name": "reflectance[band=red]", "eo:common_name": "red"},
    ],
    "cube:variables": {
      "reflectance": {
        "dimensions": ["y", "x", "band"],
        "type": "data",
      }
    },
    "cube:dimensions": {
      "band": {
        "type": "spectral",
        "description": "Spectral bands",
        "values": ["blue", "green", "red"]
      }
    }
  }
}
```

##### Bands in multiscales groups

For multi-resolution data organized using the [multiscales convention](https://github.com/zarr-conventions/multiscales), bands are organized within resolution groups hosting data variables.

The hierarchical structure of a multiscales Zarr store looks like this:

```console
dataset.zarr/
      │
      ├─► measurements/
      │   │
      │   └─► reflectance/         (multiscales group)
      │        │
      │        ├─► r10m/           (resolution level)
      │        │    ├─► b02/       (array)
      │        │    ├─► b03/       (array)
      │        │    └─► b04/       (array)
      │        │
      │        ├─► r20m/           (resolution level)
      │        │    └─► ...
      │        │
      │        ├─► r60m/           (resolution level)
      │        │    └─► ...
      │
      └─► quality/
```

The `bands` array SHALL reference individual bands, and the `name` field SHOULD follow the same pattern as in the "Unique band per data variable" and "Multiple bands per data variable" cases, depending on how bands are organized within each resolution group.
The key point is that the resolution group do not appear directly in the metadata but is implicitly hidden behind the asset href. It is up to the client to construct and select the correct path to access individual arrays based on the [layout](https://github.com/zarr-conventions/multiscales?tab=readme-ov-file#layout) given in the multiscales convention.

```json
"assets": {
    "reflectance": {
      "href": "s3://bucket/path/data.zarr/measurements/reflectance",
      "type": "application/vnd+zarr; version=3; profile=multiscales",
      "title": "Surface Reflectance",
      "gsd": 10,
      "bands": [
        {
          "name": "b01",
          "eo:common_name": "coastal",
          "description": "Coastal aerosol (band 1)",
          "gsd": 60
        },
        {
          "name": "b02",
          "eo:common_name": "blue",
          "description": "Blue (band 2)",
        },
        {
          "name": "b03",
          "eo:common_name": "green",
          "description": "Green (band 3)",
        },
        {
          "name": "b04",
          "eo:common_name": "red",
          "description": "Red (band 4)",
        },
        {
          "name": "b05",
          "eo:common_name": "rededge",
          "description": "Red edge 1 (band 5)",
          "gsd": 20
        },
        {
          "name": "b06",
          "eo:common_name": "rededge",
          "description": "Red edge 2 (band 6)",
          "gsd": 20
        },
        {
          "name": "b07",
          "eo:common_name": "rededge",
          "description": "Red edge 3 (band 7)",
          "gsd": 20
        },
        {
          "name": "b8A",
          "eo:common_name": "nir08",
          "description": "NIR 2 (band 8A)",
          "gsd": 20
        },
        {
          "name": "b08",
          "eo:common_name": "nir",
          "description": "NIR 1 (band 8)",
        },
        {
          "name": "b09",
          "eo:common_name": "nir09",
          "description": "NIR 3 (band 9)",
          "gsd": 60
        },
        {
          "name": "b11",
          "eo:common_name": "swir16",
          "description": "SWIR 1 (band 11)",
          "gsd": 20
        },
        {
          "name": "b12",
          "eo:common_name": "swir22",
          "description": "SWIR 2 (band 12)",
          "gsd": 20
        }
      ],
      "roles": ["data", "reflectance"]
    }
}
```

### Link Relationships and Types

#### Store Link Relationship

The Zarr store SHOULD be referenced with a link using the `"store"` relationship. This is required for clients to discover the root of the Zarr hierarchy and locate the related assets.

```json
"links": [
  {
    "rel": "store",
    "href": "s3://bucket/path/data.zarr",
    "type": "application/vnd+zarr; version=3",
    "title": "Zarr Store"
  }
]
```

> [!IMPORTANT] This best practice assumes that all assets referenced in the STAC object are contained within the same Zarr store.

##### Virtual Zarr Stores

For virtual Zarr stores (created using tools like Kerchunk or VirtualiZarr), the link with `rel: store` SHOULD point to the entrypoint of the virtual Zarr store. This may be a reference file (e.g., JSON file, Parquet file) or another format (e.g., an icechunk store) that provides the mapping to the underlying data.

#### Media Type

The media type for Zarr should include version information as a parameter:

- Zarr v2: `"application/vnd.zarr; version=2"`
- Zarr v3: `"application/vnd.zarr; version=3"`

### Link Templates

#### Link Template Relationship

[Link Templates](https://github.com/stac-extensions/link-templates/) CAN be used to refer to data variables contained by a Zarr group that is referenced by an asset:

```json
"linkTemplates": [
  {
    "rel": "data-variable",
    "title": "r10m",
    "uriTemplate": "s3://bucket/path/data.zarr/group/{band}",
    "variables": {
      "band": {
        "description": "...",
        "type": "string",
        "enum": [
          "B02",
          "B03",
          "B04",
        ]
      }
    }
  }
]
```

## STAC Extensions Requirements

1. **[Datacube extension](https://github.com/stac-extensions/datacube) >=v2.0.0 SHOULD be used to describe variables and dimensions when relevant**

   It is used to describe multidimensional structure for groups containing multiple variables, use `cube:variables` and `cube:dimensions`, especially when variable arrays have more than the 2 spatial dimensions.

   - Available variables and their dimensional structure
   - Dimension extents and coordinate systems
   - Relationships between variables

2. **[Projection Extension](https://github.com/stac-extensions/projection) >=v2.0.0 SHOULD be used to describe spatial reference information**

   Key attributes to include:

   - `proj:code`, `proj:wkt2`, or `proj:projjson`
   - `proj:shape` and `proj:transform`

  If the referenced Zarr store follows the [geo-proj convention](https://github.com/zarr-conventions/geo-proj), the projection information should be consistent with the convention.

3. **[Raster Extension](https://github.com/stac-extensions/raster) >=v2.0.0 SHOULD be used for raster-like arrays**
  
   - `raster:spatial_resolution`
   - `raster:scale` and `raster:offset` (if applicable)
   - `data_type` and `nodata`

4. **[CF Extension](https://github.com/stac-extensions/cf) SHOULD be used for climate and forecast data following CF conventions**

   For data following CF conventions (Climate and Forecast), relevant attributes like `standard_name`, `units`, and `cell_methods` should be represented in STAC metadata.

## Use Cases

### Scene-Based Zarr Stores (EOPF Products)

**Context**: Earth Observation Processing Framework (EOPF) products organize data in hierarchical Zarr stores with multiple resolution groups. Example: Sentinel-2 Level-2A data.

**Store Structure**:

```console
/
├── conditions/
├── measurements/
│   └── reflectance/
│       ├── r10m/
│       │   ├── b02 (10980, 10980) uint16
│       │   ├── b03 (10980, 10980) uint16
│       │   ├── b04 (10980, 10980) uint16
│       │   ├── b08 (10980, 10980) uint16
│       │   ├── x (10980,) int64
│       │   └── y (10980,) int64
│       ├── r20m/
│       └── r60m/
└── quality/
```

#### Resolution based group representation

In this [example](examples/S2A_MSIL2A_20251008T100041_N0511_R122_T32TQM_20251008T122613.json), each resolution group (e.g., `r10m`, `r20m`, `r60m`) is represented as a separate asset in the STAC Item. Each asset points to the corresponding Zarr group containing the data arrays for that resolution.

**Key Points**:

- Each asset href points to a specific resolution group
- Common properties (data_type, resolution) are at asset level
- Band-specific metadata is in the `bands` array
- Variable names match array names in the Zarr group
- Clients can easily discover and access data at different resolutions
- This pattern is useful when users need to work with specific resolutions independently

**Accessing Individual Arrays**:

The `name` field in the `bands` array plays a crucial role for programmatic data access. It should be constructed so that concatenating (path joining) the asset `href` and the band `name` results in the correct Zarr URL:

```python
import zarr

asset_href = "s3://bucket/S2B_MSIL2A_20251006T123309.zarr/measurements/reflectance/r10m"
band_name = "b04"
# Access individual array
red = zarr.open_array(asset_href + "/" + band_name)
```

#### Multiscale reflectance group representation

The [single multiscales asset example](examples/S2A_MSIL2A_20251008T100041_N0511_R122_T32TQM_20251008T122613_single_asset.json) represents the `reflectance` group at multiple resolutions.

**Key Points**:

- The store link provides the root Zarr location
- A single asset represents the entire multiscales group
- The `profile` parameter indicates multiscales organization
- Common properties (data_type, resolution) are at asset level
- Band-specific metadata is in the `bands` array with overrides for band specific properties (e.g., `gsd`)
- Variable names match array names in the Zarr multiscales groups
- Clients can discover and access data at different resolutions by navigating the multiscales hierarchy

**Accessing Individual Arrays**:

The `name` field in the `bands` array is used to identify individual bands. Clients need to construct the correct path to access specific arrays based on the multiscales layout.

```python
import zarr
import json

# Asset href from STAC Item
asset_href = "s3://bucket/S2A_MSIL2A_20251008T100041.zarr/measurements/reflectance"

# Open the multiscales group
group = zarr.open_group(asset_href, mode='r')

# Read the multiscales metadata
multiscales_metadata = group.attrs.get('multiscales', {})
layout = multiscales_metadata.get('layout', [])

# Discover available resolution levels
print("Available resolution levels:")
for level in layout:
    group_name = level['group']
    factors = level.get('factors', 'N/A')
    print(f"  - {group_name}: factors={factors}")

# Select a specific resolution level (e.g., the first level)
target_level = layout[0]['group']  # e.g., "r10m" or "0"

# Access a specific band at this resolution
band_name = "b04"  # Red band
array_path = f"{target_level}/{band_name}"
red_array = group[array_path]

print(f"\nAccessing {band_name} at resolution level {target_level}:")
print(f"  Shape: {red_array.shape}")
print(f"  Dtype: {red_array.dtype}")

# Read data (example: read a subset)
data_subset = red_array[0:100, 0:100]
```

Alternatively, for direct array access when the resolution level is known:

```python
import zarr

# Construct full path to specific array
asset_href = "s3://bucket/S2A_MSIL2A_20251008T100041.zarr/measurements/reflectance"
resolution_level = "r10m"  # From multiscales layout
band_name = "b04"

# Open array directly
red = zarr.open_array(f"{asset_href}/{resolution_level}/{band_name}", mode='r')
```

### CF-Compliant Climate and Weather Data

**Context**: Climate model outputs and weather simulations often follow CF (Climate and Forecast) conventions and contain numerous variables with complex dimensional structures.

**Store Structure**:

```console
s3://bucket/ICON_d3hp003.zarr/
└── P1D_mean_z5_atm/  (single group)
    ├── clivi (425, 12288) float32
    ├── clt (425, 12288) float32
    ├── hfls (425, 12288) float32
    ├── pr (425, 12288) float32
    ├── tas (425, 12288) float32
    ├── ta (425, 30, 12288) float32  # 3D with pressure levels
    ├── time (425) int64 # seconds since some reference date
    ├── cell (12288) int32
    └── pressure (30) float32
```

The [example STAC Item](examples/ICON_d3hp003_cf.json) represents a Zarr store containing multiple CF-compliant variables with different dimensional structures.

**Key Points**:

- Single asset represents a Zarr group with multiple CF variables
- Multiple dimension types: temporal, spatial (non-standard grid), vertical
- Variables have different dimensional structures (2D vs 3D)
- CF standard names are preserved using a `cf:` prefix
- The `bands` array provides human-readable variable descriptions
- Non-geographic spatial reference system (HEALPix) is indicated

### Virtual Zarr Stores

**Context**: Tools like [Kerchunk](https://fsspec.github.io/kerchunk/) and [VirtualiZarr](https://virtualizarr.readthedocs.io/) create virtual Zarr stores that reference existing data files (e.g., NetCDF, HDF5, GRIB) without copying data. This enables Zarr-like access to legacy formats through a reference system that maps to the original data locations.

Virtual Zarr stores can be represented in both STAC Items and Collections, following the same patterns described in the [STAC Object Type](#stac-object-type) section:

- **STAC Item**: Appropriate when the virtual Zarr store represents a single logical dataset (e.g., a single time slice or scene)
- **STAC Collection**: Appropriate when organizing multiple related virtual Zarr stores (e.g., time series) or when a single virtual Zarr store contains a time dimension spanning the entire collection

The [example STAC Collection](examples/CMIP6_ScenarioMIP_NCAR_CESM2_collection.json) represents a virtual Zarr store stored as icechunk for a particular model (spanning multiple source NetCDF files)

The [example STAC Item](examples/CMIP6_ScenarioMIP_NCAR_CESM2_item.json) represents a virtual Zarr store stored as kerchunk for a particular NetCDF file.

**Key Points**:

- The link with `rel: store` SHOULD point to the entrypoint of the virtual Zarr store (e.g., a reference file or an icechunk store)
- Assets with role `"virtual"` reference the Zarr groups accessible through the virtual store
- Use [ZEP 8](https://github.com/jbms/url-pipeline) to construct an href pointing to a subgroup within a virtual Zarr store (this is not yet widely supported by tooling)
- Optionally, implementers MAY include assets with role `"source"` to reference the source file or the bucket/directory containing the underlying data files
- The virtual nature of the store is typically abstracted from users, who interact with it like a native Zarr store
- For Collections, the same principles apply: include a `rel: store` link at the Collection level if all Items share the same virtual store, or at the Item level if each Item has its own virtual store
