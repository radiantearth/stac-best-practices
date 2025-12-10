# STAC Best Practices

> [!WARNING]
> This repository is an early version based on the stac-spec best practices and is currently getting an overhaul.
>
> Feedback is always welcome, ideally via GitHub issues or pull requests.

This repository contains the STAC Best Practices. 

This repository makes a number of recommendations for creating real world SpatioTemporal Asset Catalogs. None of them 
are required to meet the core specification, but following these practices will make life easier for client tooling
and for users. They come about from practical experience of implementors and introduce a bit more 'constraint' for
those who are creating STAC objects representing their data or creating tools to work with STAC. 

While the current goal of the core is to remain quite flexible and simple to meet a wide variety of use cases,
in time some of these may evolve to become part of the core specification.

## STAC Specification

- [Web Best Practices](best-practices-web.md)
- [Item Best Practices](best-practices-item.md)
- [Asset and Link Best Practices](best-practices-asset-and-link.md)
- [Catalog and Collection Best Practices](best-practices-catalog-and-collection.md)
- [Metadata and Extension Best Practices](best-practices-metadata-and-extension.md)
- [Zarr Best Practices](best-practices-zarr.md)

## STAC API Specification

Not established yet.

## Implementation Guide

> [!WARNING]  
> The implementation guide is in its early stages and is not complete yet.

- [Implementation Guide](implementation-guide.md)

## Extension-Specific Best Practices

The following best practices are endorsed by the community for each extension individually:

- [Machine Learning Model Extension](https://github.com/stac-extensions/mlm/blob/main/best-practices.md)
- [Projection Extension](https://github.com/stac-extensions/projection/blob/main/README.md#best-practices)
