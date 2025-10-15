# STAC Zarr and N-Dimensional Array Best Practices

## Table of Contents

- [Introduction](#introduction)
- [Glossary](#glossary)
- [General Rules](#general-rules)
  - [Asset Organization](#asset-organization)
  - [Link Relationships](#link-relationships)
  - [Metadata Requirements](#metadata-requirements)
- [Required Extensions](#required-extensions)
- [Use Cases](#use-cases)
  - [Scene-Based Zarr Stores (EOPF Products)](#scene-based-zarr-stores-eopf-products)
  - [CF-Compliant Climate and Weather Data](#cf-compliant-climate-and-weather-data)
  - [Virtual Zarr with Reference Files](#virtual-zarr-with-reference-files)
- [Asset Discovery and Access Patterns](#asset-discovery-and-access-patterns)
- [Credits](#credits)

## Introduction

This document provides best practices for representing Zarr stores and other n-dimensional array formats in STAC. These guidelines address:

- Asset hierarchy and organization for Zarr groups and arrays
- Discovery patterns for variables within multidimensional stores
- Integration with datacube extension for variable and dimension metadata
- Use of link relationships to reference Zarr stores
- Asset metadata for multi-resolution and hierarchical data structures

The following specifications and extensions are discussed in this document:

- [STAC v1.1](https://github.com/radiantearth/stac-spec/tree/v1.1.0)
- [Datacube Extension v2.x](https://github.com/stac-extensions/datacube)
- [Raster Extension v2.x](https://github.com/stac-extensions/raster)
- [Electro Optical (EO) Extension v2.x](https://github.com/stac-extensions/eo)
- [Projection Extension v2.x](https://github.com/stac-extensions/projection)

## Glossary

### Zarr Terminology

See the [Zarr specification](https://zarr-specs.readthedocs.io/en/latest/v3/core/#concepts-and-terminology) for complete terminology.

Key terms:

- **Zarr Store**: A storage system that can hold a Zarr hierarchy (e.g., a directory, zip file, or object store prefix)
- **Group**: A node including the root node in a Zarr hierarchy that can contain arrays and/or other groups
- **Array**: A multidimensional array with chunked storage
- **Chunk**: A contiguous region of an array that is stored as a single object
- **Consolidated Metadata**: A single metadata document containing the metadata for all arrays and groups in a store

### Xarray/NetCDF/CF-Specific Concepts

- **Variable**: A named data array within a Zarr group, typically representing a measured or derived quantity
- **Dimension**: An axis of a multidimensional array (e.g., x, y, time, band)
- **Coordinate Variable**: An array that provides coordinate values along a dimension
- **Data Variable**: An array containing measured or derived data values

## General Rules

### Asset Organization

1. **A Zarr asset SHALL reference a group containing one or more arrays or groups**
   
   This is equivalent to an xarray Dataset or an xarray DataTree.

2. **The Zarr store SHOULD be referenced with a link using the `"store"` relationship**
   
   ```json
   "links": [
     {
       "rel": "store",
       "href": "s3://bucket/path/data.zarr",
       "type": "application/vnd+zarr; version=2",
       "title": "Zarr Store"
     }
   ]
   ```

3. **Assets cannot reference individual arrays within the store**
   
   Assets may point to:
   - The root group
   - Sub groups
   
   The appropriate level depends on how users will access the data.

4. **Zarr STAC Extension SHOULD align with Zarr conventions (that are currently being defined) to clearly indicate the data model that best represents the asset**
   
   - `"multi-resolution"`: Reference to a multi-resolution datatree
   - `"single-resolution"`: Reference to a single resolution dataset

### Link Relationships

**Store Link Relationship**

A new link relationship `"store"` is used to reference Zarr stores from Collections or Items:

```json
"links": [
  {
    "rel": "store",
    "href": "s3://bucket/path/data.zarr",
    "type": "application/vnd+zarr; version=2",
    "title": "Zarr Store"
  }
]
```

**TODO**: Propose this relationship for inclusion in the storage STAC extension.

**Media Type**

The media type for Zarr should include version information as a parameter:

- Zarr v2: `"application/vnd+zarr; version=2"`
- Zarr v3: `"application/vnd+zarr; version=3"`

This follows the pattern established by Cloud Optimized GeoTIFF:
```json
"type": "image/tiff; application=geotiff; profile=cloud-optimized"
```

### Metadata Requirements

1. **The datacube extension SHOULD be used to describe variables and dimensions when relevant**
   
   For groups containing multiple variables, use `cube:variables` and `cube:dimensions` to document:
   - Available variables and their dimensional structure
   - Dimension extents and coordinate systems
   - Relationships between variables

2. **Band information SHOULD be provided for spectral/multi-channel data**
   
   The `bands` array should contain keys corresponding to data variable names, with values providing band-specific metadata.

3. **CF convention attributes SHOULD be preserved when applicable**
   
   For data following CF conventions (Climate and Forecast), relevant attributes like `standard_name`, `units`, and `cell_methods` should be represented in STAC metadata.

   TODO: to be refined with the community.

## Required Extensions

For Zarr assets, the following extensions are typically required:

- **Datacube Extension (v2.x)**: To describe multidimensional structure
  - `cube:dimensions`: Define coordinate dimensions
  - `cube:variables`: Describe data variables within groups

- **Projection Extension (v2.x)**: For georeferenced arrays
  - `proj:code`, `proj:wkt2`, or `proj:projjson`
  - `proj:shape` and `proj:transform`

- **Raster Extension (v2.x)**: For raster-like arrays
  - `raster:spatial_resolution`
  - `raster:scale` and `raster:offset` (if applicable)
  - `data_type` and `nodata`

## Use Cases

### Scene-Based Zarr Stores (EOPF Products)

**Context**: Earth Observation Processing Framework (EOPF) products organize data in hierarchical Zarr stores with multiple resolution groups. Example: Sentinel-2 Level-2A data.

**Store Structure**:
```
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

**STAC Representation**:

```json
{
  "type": "Feature",
  "stac_version": "1.1.0",
  "id": "S2B_MSIL2A_20251006T123309",
  "properties": {
    "datetime": "2025-10-06T12:33:09Z"
  },
  "links": [
    {
      "rel": "store",
      "href": "s3://bucket/S2B_MSIL2A_20251006T123309.zarr",
      "type": "application/vnd+zarr; version=2",
      "title": "Sentinel-2 L2A Zarr Store"
    }
  ],
  "assets": {
    "measurements_r10m": {
      "href": "s3://bucket/S2B_MSIL2A_20251006T123309.zarr/measurements/reflectance/r10m",
      "type": "application/vnd+zarr; version=2",
      "roles": ["group", "data"],
      "raster:spatial_resolution": 10,
      "data_type": "uint16",
      "nodata": 0,
      "raster:scale": 0.0001,
      "raster:offset": -0.1,
      "proj:code": "EPSG:32628",
      "proj:shape": [10980, 10980],
      "cube:dimensions": {
        "x": {
          "type": "spatial",
          "axis": "x",
          "extent": [300000, 409800]
        },
        "y": {
          "type": "spatial",
          "axis": "y",
          "extent": [0, 109800]
        }
      },
      "cube:variables": {
        "b02": {
          "dimensions": ["y", "x"],
          "type": "data",
          "unit": null
        },
        "b03": {
          "dimensions": ["y", "x"],
          "type": "data",
          "unit": null
        },
        "b04": {
          "dimensions": ["y", "x"],
          "type": "data",
          "unit": null
        },
        "b08": {
          "dimensions": ["y", "x"],
          "type": "data",
          "unit": null
        }
      },
      "bands": [
        {
          "name": "b02",
          "eo:common_name": "blue",
          "description": "Blue band (490 nm)"
        },
        {
          "name": "b03",
          "eo:common_name": "green",
          "description": "Green band (560 nm)"
        },
        {
          "name": "b04",
          "eo:common_name": "red",
          "description": "Red band (665 nm)"
        },
        {
          "name": "b08",
          "eo:common_name": "nir",
          "description": "NIR band (842 nm)"
        }
      ]
    }
  }
}
```

**Key Points**:
- The store link provides the root Zarr location
- Assets point to specific resolution groups
- Common properties (data_type, resolution) are at asset level
- Band-specific metadata is in the `bands` array
- Variable names match array names in the Zarr group

### CF-Compliant Climate and Weather Data

**Context**: Climate model outputs and weather simulations often follow CF (Climate and Forecast) conventions and contain numerous variables with complex dimensional structures.

**Store Structure**:
```
s3://bucket/ICON_d3hp003.zarr/
└── P1D_mean_z5_atm/  (single group)
    ├── clivi (425, 12288) float32
    ├── clt (425, 12288) float32
    ├── hfls (425, 12288) float32
    ├── pr (425, 12288) float32
    ├── tas (425, 12288) float32
    ├── ta (425, 12288, 30) float32  # 3D with pressure levels
    ├── time (425) datetime64[ns]
    ├── cell (12288) int32
    └── pressure (30) float32
```

**STAC Representation**:

```json
{
  "type": "Feature",
  "stac_version": "1.1.0",
  "id": "ICON_d3hp003_P1D_mean_z5_atm",
  "properties": {
    "start_datetime": "2020-01-02T00:00:00Z",
    "end_datetime": "2021-03-01T00:00:00Z"
  },
  "links": [
    {
      "rel": "store",
      "href": "s3://bucket/ICON_d3hp003.zarr",
      "type": "application/vnd+zarr; version=2",
      "title": "ICON Model Output Zarr Store"
    }
  ],
  "assets": {
    "P1D_mean_z5_atm": {
      "href": "s3://bucket/ICON_d3hp003.zarr/P1D_mean_z5_atm",
      "type": "application/vnd+zarr; version=2",
      "roles": ["group", "data"],
      "proj:shape": [425, 12288, 30, 5, 3],
      "cube:dimensions": {
        "time": {
          "type": "temporal",
          "description": "time",
          "extent": ["2020-01-02T00:00:00Z", "2021-03-01T00:00:00Z"],
          "step": "P1D",
          "unit": "seconds since 2020-01-01T00:00:00"
        },
        "cell": {
          "type": "spatial",
          "axis": "cell",
          "description": "cell index in HEALPix grid",
          "extent": [0, 12287],
          "step": 1,
          "reference_system": "HEALPix"
        },
        "pressure": {
          "type": "vertical",
          "axis": "z",
          "description": "pressure level",
          "extent": [5.0, 100000.0],
          "unit": "Pa"
        }
      },
      "cube:variables": {
        "clivi": {
          "dimensions": ["time", "cell"],
          "type": "data",
          "unit": "kg m-2",
          "description": "cloud ice path"
        },
        "clt": {
          "dimensions": ["time", "cell"],
          "type": "data",
          "unit": "m2 m-2",
          "description": "total cloud cover"
        },
        "hfls": {
          "dimensions": ["time", "cell"],
          "type": "data",
          "unit": "W m-2",
          "description": "surface upward latent heat flux"
        },
        "pr": {
          "dimensions": ["time", "cell"],
          "type": "data",
          "unit": "kg m-2 s-1",
          "description": "precipitation flux"
        },
        "tas": {
          "dimensions": ["time", "cell"],
          "type": "data",
          "unit": "K",
          "description": "near-surface air temperature"
        },
        "ta": {
          "dimensions": ["time", "cell", "pressure"],
          "type": "data",
          "unit": "K",
          "description": "air temperature"
        }
      },
      "bands": [
        {
          "name": "clivi",
          "description": "cloud ice path",
          "cf:standard_name": "atmosphere_mass_content_of_cloud_ice",
          "unit": "kg m-2"
        },
        {
          "name": "clt",
          "description": "total cloud cover",
          "cf:standard_name": "cloud_area_fraction",
          "unit": "m2 m-2"
        },
        {
          "name": "hfls",
          "description": "surface upward latent heat flux",
          "cf:standard_name": "surface_upward_latent_heat_flux",
          "unit": "W m-2"
        },
        {
          "name": "pr",
          "description": "precipitation flux",
          "cf:standard_name": "precipitation_flux",
          "unit": "kg m-2 s-1"
        },
        {
          "name": "tas",
          "description": "near-surface air temperature",
          "cf:standard_name": "air_temperature",
          "unit": "K"
        },
        {
          "name": "ta",
          "description": "air temperature",
          "cf:standard_name": "air_temperature",
          "unit": "K"
        }
      ]
    }
  }
}
```

**Key Points**:
- Single asset represents an entire xarray Dataset
- Multiple dimension types: temporal, spatial (non-standard grid), vertical
- Variables have different dimensional structures (2D vs 3D)
- CF standard names are preserved using a `cf:` prefix
- The `bands` array provides human-readable variable descriptions
- Non-geographic spatial reference system (HEALPix) is indicated

### Virtual Zarr with Reference Files

**Context**: Kerchunk and similar tools create virtual Zarr stores that reference existing data files (e.g., NetCDF) without copying data. This enables Zarr-like access to legacy formats.

**STAC Representation**:

```json
{
  "type": "Feature",
  "stac_version": "1.1.0",
  "id": "CMIP6_ScenarioMIP_NCAR_CESM2",
  "properties": {
    "start_datetime": "2015-01-01T00:00:00Z",
    "end_datetime": "2024-12-31T23:59:59Z"
  },
  "links": [
    {
      "rel": "store",
      "href": "https://storage.example.com/kerchunk/CMIP6_CESM2_rlus_kr1.0.json",
      "type": "application/json+zstd",
      "title": "Kerchunk Reference File"
    }
  ],
  "assets": {
    "reference": {
      "href": "https://storage.example.com/kerchunk/CMIP6_CESM2_rlus_kr1.0.json",
      "type": "application/json+zstd",
      "title": "Kerchunk reference file",
      "roles": ["reference", "data"],
      "description": "Virtual Zarr store referencing NetCDF files"
    },
    "source_data": {
      "href": "https://storage.example.com/data/CMIP6/rlus_day_CESM2-WACCM_ssp585_gn_20150101-20241231.nc",
      "type": "application/netcdf",
      "title": "Original NetCDF file",
      "roles": ["source"]
    }
  }
}
```

**Key Points**:
- The store link points to the reference file
- Assets include both the reference file and source data
- Media type indicates the reference format (e.g., JSON with zstd compression)
- Role `"reference"` indicates virtual/indirect data access
- Role `"source"` indicates the underlying data files

## Asset Discovery and Access Patterns

### Accessing Zarr Subgroups

For hierarchical stores, provide assets at meaningful organizational levels:

```json
"assets": {
  "measurements_r10m": {
    "href": "s3://bucket/data.zarr/measurements/reflectance/r10m",
    "roles": ["group", "data"],
    "description": "10m resolution measurements"
  },
  "measurements_r20m": {
    "href": "s3://bucket/data.zarr/measurements/reflectance/r20m",
    "roles": ["group", "data"],
    "description": "20m resolution measurements"
  }
}
```

### Multi-Resolution and Pyramid Data

EARLY DRAFT: to be updated with multiscale conventions in Zarr.

For data with multiple resolutions (pyramids), each resolution level should be a separate asset:

```json
"assets": {
  "data_level_0": {
    "href": "s3://bucket/data.zarr/0",
    "roles": ["data"],
    "raster:spatial_resolution": 10,
    "proj:shape": [10980, 10980]
  },
  "data_level_1": {
    "href": "s3://bucket/data.zarr/1",
    "roles": ["overview"],
    "raster:spatial_resolution": 20,
    "proj:shape": [5490, 5490]
  },
  "data_level_2": {
    "href": "s3://bucket/data.zarr/2",
    "roles": ["overview"],
    "raster:spatial_resolution": 40,
    "proj:shape": [2745, 2745]
  }
}
```
