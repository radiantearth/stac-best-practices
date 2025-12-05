# STAC Best Practices

> [!WARNING]
> This repository is a very early version and not officially endorsed by the STAC PSC yet.
> We plan to also move the best practices from the official stac-spec and stac-api-spec repositories over to this repository in the future.
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

## Contributing

We use [markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2) to lint our markdown files.
To run it locally:

```sh
npm install markdownlint-cli2 --global
markdownlint-cli2
```

To auto-fix some lint errors:

```sh
markdownlint-cli2 --fix
```

We use [prek](https://prek.j178.dev/) to run our linting tools in CI.
If you'd like to use it locally, [install it](https://github.com/j178/prek?tab=readme-ov-file#installation), then:

```sh
prek run --all-files
```

To automatically run **prek** on your commits:

```sh
prek install
```
