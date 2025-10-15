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
- **Consolidated Metadata**: A single metadata document containing the metadata for all arrays and groups that are children of a given group.

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

4. **Band names SHALL BE relative to the asset group and point to an array within that group**

    The band name shall be used as a relative path for accessing an array to point to a specific array within the group linked in the asset `href`.
    Implicitly, the band has the "data" role.

5. **Zarr STAC Extension SHOULD align with Zarr conventions (that are currently being defined) to clearly indicate the data model that best represents the asset**
   
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

4. **A specific `profile=multiscales` parameter in the content-type SHOULD be used for assets representing multi-resolution data**

    Example:
    ```json
    "type": "application/vnd+zarr; version=3; profile=multiscales"
    ```
  
    This follows the pattern established by Cloud Optimized GeoTIFF:
    ```json
    "type": "image/tiff; application=geotiff; profile=cloud-optimized"
    ```

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

1 big reflectance STAC Representation**

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
    "reflectance": {
      "gsd": 10,
      "href": "https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202510-s02msil2a-eu/14/products/cpm_v256/S2C_MSIL2A_20251014T142151_N0511_R096_T25WET_20251014T161521.zarr/measurements/reflectance/",
      "type": "application/vnd+zarr",
      "proj:epsg": 32625,
      "proj:shape": [10980, 10980],
      "bands": [
        {
          "name": "r60m/b01",
          "common_name": "coastal",
          "description": "Coastal aerosol (band 1)",
          "center_wavelength": 0.443,
          "full_width_half_max": 0.027,
          "proj:shape": [1980, 1980]
        },
        {
          "name": "r10m/b02",
          "common_name": "blue",
          "description": "Blue (band 2)",
          "center_wavelength": 0.49,
          "full_width_half_max": 0.098
        },
        {
          "name": "r10m/B03",
          "common_name": "green",
          "description": "Green (band 3)",
          "center_wavelength": 0.56,
          "full_width_half_max": 0.045
        },
        {
          "name": "r10m/B04",
          "common_name": "red",
          "description": "Red (band 4)",
          "center_wavelength": 0.665,
          "full_width_half_max": 0.038
        },
        {
          "name": "r20m/B05",
          "common_name": "rededge",
          "description": "Red edge 1 (band 5)",
          "center_wavelength": 0.704,
          "full_width_half_max": 0.019,
          "proj:shape": [5940, 5940]
        },
        {
          "name": "r20m/B06",
          "common_name": "rededge",
          "description": "Red edge 2 (band 6)",
          "center_wavelength": 0.74,
          "full_width_half_max": 0.018,
          "proj:shape": [5940, 5940]
        },
        {
          "name": "r20m/B07",
          "common_name": "rededge",
          "description": "Red edge 3 (band 7)",
          "center_wavelength": 0.783,
          "full_width_half_max": 0.028,
          "proj:shape": [5940, 5940]
        },
        {
          "name": "r20m/B8A",
          "common_name": "nir08",
          "description": "NIR 2 (band 8A)",
          "center_wavelength": 0.865,
          "full_width_half_max": 0.033,
          "proj:shape": [5940, 5940]
        },
        {
          "name": "r10m/B08",
          "common_name": "nir",
          "description": "NIR 1 (band 8)",
          "center_wavelength": 0.842,
          "full_width_half_max": 0.145
        },
        {
          "name": "r60m/B09",
          "common_name": "nir09",
          "description": "NIR 3 (band 9)",
          "center_wavelength": 0.945,
          "full_width_half_max": 0.026,
          "proj:shape": [1980, 1980]
        },
        {
          "name": "r20m/B11",
          "common_name": "swir16",
          "description": "SWIR 1 (band 11)",
          "center_wavelength": 1.61,
          "full_width_half_max": 0.143,
          "proj:shape": [5940, 5940]
        },
        {
          "name": "r20m/B12",
          "common_name": "swir22",
          "description": "SWIR 2 (band 12)",
          "center_wavelength": 2.19,
          "full_width_half_max": 0.242,
          "proj:shape": [5940, 5940]
        }
      ],
      "roles": ["reflectance", "multiscales"],
      "title": "Surface Reflectance"
    }
  },
  "linkTemplates": [
    {
      "rel": "data-variable",
      "title": "store",
      "uriTemplate": "https://objects.eodc.eu:443/e05ab01a9d56408d82ac32d69a5aae2a:202510-s02msil2a-eu/14/products/cpm_v256/S2C_MSIL2A_20251014T142151_N0511_R096_T25WET_20251014T161521.zarr/measurements/reflectance/{resolution}/{band}",
      "variables": {
        "resolution": {
          "description": "resolution",
          "type": "string",
          "enum": ["r10m", "r20m", "r60m"]
        },
        "band": {
          "description": "band name",
          "type": "string",
          "enum": ["b01", "b02", "b03", "b04", "b05", "b06", "b07", "b8A", "b08", "b09", "b11", "b12"]
        }
      }
    }
  ]
}
```

`linkTemplates` provides a way to construct URLs for accessing individual arrays programmatically. This is not necessarily required but can be very useful for clients.

**Key Points**:
- The store link provides the root Zarr location
- Assets point to specific resolution groups
- Common properties (data_type, resolution) are at asset level
- Band-specific metadata is in the `bands` array
- Variable names match array names in the Zarr group

**Accessing Individual Arrays**:

The `name` field in the `bands` array plays a crucial role for programmatic data access. It should be constructed so that concatenating (path joining) the asset `href` and the band `name` results in the correct Zarr URL:

```python
import zarr

asset_href = "s3://bucket/S2B_MSIL2A_20251006T123309.zarr/measurements/reflectance/r10m"
band_name = "b04"

# Access individual array
red = zarr.open_array(asset_href + "/" + band_name)
```

This pattern enables applications to:
1. Discover available variables through the STAC metadata
2. Construct the correct path to access specific arrays
3. Load data without needing to parse the entire Zarr hierarchy

For xarray users, this would look like:

```python
import xarray as xr

dataset = xr.open_zarr(asset_href)
red = dataset["b04"].chunk({}).persist()
```

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
    ├── time (425) int64 # seconds since some reference date
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
      "bbox": [
        -180.0,
        -90.0,
        180.0,
        90.0
      ],
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
          "unit": "kg m-2",
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
    "description": "10m resolution measurements"
  },
  "measurements_r20m": {
    "href": "s3://bucket/data.zarr/measurements/reflectance/r20m",
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
