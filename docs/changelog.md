# Changelog

Changelog for the documentation see: [The Landscaping System Docs on Github](https://github.com/ludicdrive/landscaping-system-docs)

## Unreleased

- Feature: [Automation](automation.md?id=automation) - drive the plugin from Python, including through an AI agent
- Feature: [Allow Automation Writes](settings.md?id=allow-automation-writes) to opt into scripted Level modifications
- Feature: `tiles_processed` now reports the tiles that actually received data instead of the tiles that were walked, and an import that reaches no tile is reported as a failure with a reason
- Bugfix: creating Actors, Splines and Blueprints used a stale tile selection instead of the current one
- Bugfix: scripted runs no longer save the Level implicitly. The interactive flow saves in a few places and those saves are now suppressed while automating, so a script cannot persist anything on its own
- Bugfix: a projected WKT without a matching EPSG code is accepted instead of rejected, and EPSG codes are validated through PROJ instead of against a numeric range
- Bugfix: a CRS that has no EPSG code is now shown by name with a *(file CRS, no EPSG code)* note in the Landscaping System Tab and DTM import dialog, and its tooltip states the current CRS, instead of an *EPSG:0 / no CRS set* error
- Bugfix: a geographic or non-metric CRS is refused as a Level CRS with an actionable error rather than importing into the wrong place
- Docs: [Allowed Vector File Types](settings.md?id=allowed-vector-file-types), [Extended CRS Compatibility](settings.md?id=extended-crs-compatibility) and [Coordinate reference systems](settings.md?id=coordinate-reference-systems)

## Version 8.4

- Feature: [Seamless horizontal Tiling](heights.md?id=seamless-horizontal-tiling)
- Bugfix: Mapbox import when file import is selected but no file loaded

## Version 8.3

- Feature: UE 5.7 compatibility

## Version 8.2

- Feature: UE 5.7 (Preview) compatibility
- Bugfix: pre-select color channel for weightmap noise texture

## Version 8.1

- Bugfix: save generated weightmaps in UE 5.6

## Version 8.0

Initial version of **Landscaping System**

## Version 7.x and older

For the changelog of Landscaping until version 7.x see [Landscaping Docs](https://jorop.github.io/landscaping-docs/#/changelog?id=changelog)