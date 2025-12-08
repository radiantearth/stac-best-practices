# Implementation Guide

## Table of Contents

- [How to Differentiate STAC Files](#how-to-differentiate-stac-files)
- [Mixing STAC Versions](#mixing-stac-versions)


## How to Differentiate STAC Files

Any tool that crawls a STAC implementation or encounters a STAC file in the wild needs a clear way to determine if it is an Item, 
Collection or Catalog. As of 1.0.0 this is done primarily
with the `type` field, and secondarily in Items with `stac_version`, or optionally with the `rel` of the link to it.

```shell
if type is 'Collection'
  => Collection
else if type is 'Catalog'
  => Catalog
else if type is 'Feature' and stac_version is defined
  => Item
else
  => Invalid (JSON)
```

When crawling a STAC implementation, one can also make use of the [relation type](catalog-spec/catalog-spec.md#relation-types
) (`rel` field) when following a link. If it is an `item` rel type then the file must be a STAC Item. If it is `child`, `parent` or
`root` then it must be a Catalog or a Collection, though the final determination between the two requires looking at the `type` field
in the Catalog or Collection JSON that it is linked to. Note that there is also a `type` field in STAC Link and Asset objects, but that
is for the Media Type, but there are not specific media types for Catalog and Collection.
See the sections on [STAC media types](commons/links.md#stac-media-types),
and [Asset media types](commons/assets.md#media-types) for more information.

In versions of STAC prior to 1.0 the process was a bit more complicated, as there was no `type` field for catalogs and collections.
See [this issue comment](https://github.com/radiantearth/stac-spec/issues/889#issuecomment-684529444) for a heuristic that works
for older STAC versions.


## Mixing STAC Versions

Although it is not recommended to mix STAC versions, it is allowed. Sometimes mixed STAC Versions are unavoidable 
when multiple catalogs or collections from different sources are combined into a single Catalog. 

Client and Server developers should be aware of this use case.
