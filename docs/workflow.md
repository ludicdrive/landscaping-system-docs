# Landscaping System workflow

The __Landscaping System__ is designed with a specific workflow in mind, which is not the only, but a very fast way to design environments from scratch.

## Start small

The __Landscaping System__ is very easy to learn, but because geospatial data can be huge and the runtime for imports can be long, it is best, to start with a small area to understand the workflow and get results quickly. 1 km² area might be enough for starters. This small level is for experimenting and trying out the artistic styles and technical parts, e.g. custom blueprints and PCG graphs. This small level should be kept to come back to and make adjustments, all of which can be rolled out in the bigger main scene.

## Workflow order

1. Import Terrain as landscape or mesh [🔗](heights.md)
2. Assign Landscape Material and create Weightmaps (optional) [🔗](landcover.md)
3. Import Satellite Data (optional) [🔗](satellite.md)
4. Import vector data [🔗](props.md)
5. Leverage Procedural Content Generation PCG [🔗](vegetation.md)

The import of the terrain has to be the starting point in any case, because there the georeference is set. If a landscape is imported as mesh (and not landscape), a landscape material is not necessary and the next step would be the import of satellite data.

## Landscaping System Tab

The __Landscaping System Tab__ reflects the workflow from top to bottom.
> :bulb: __Good to know__: By hovering with the mouse cursor over a button or label a tooltip will appear with a help text. :bulb:

### Data Source

Data source for the import operation. Currently the options `Files` and `Mapbox` is supported. Mapbox needs the [Landscaping Mapbox](https://unrealassetstore.com/product/landscaping-mapbox) extension.

### Mapbox Zoom Level

Set the Zoom Level for Mapbox heightdata imports (see [Mapbox Documentation](mapbox.md)). Only available with Mapbox Extension.