# Landscaping System workflow

The __Landscaping System__ is designed with a specific workflow in mind, which is not the only, but a very fast way to design environments from scratch.

## Start small

The __Landscaping System__ is very easy to learn, but because geospatial data can be huge and the runtime for imports can be long, it is best, to start with a small area to understand the workflow and get results quickly. 1 km² area might be enough for starters. This small level is for experimenting and trying out the artistic styles and technical parts, e.g. custom blueprints and PCG graphs. This small level should be kept to come back to and make adjustments, all of which can be rolled out in the bigger main scene.

## Workflow order

1. Import Landscape [🔗](heights.md)
2. Assign Landscape Material and create Weightmaps (optional) [🔗](landcover.md)
3. Import Satellite Data (optional) [🔗](satellite.md)
4. Import vector data [🔗](props.md)
5. Leverage Procedural Content Generation PCG [🔗](vegetation.md)

Only the import of the landscape has to be the starting point to set the georeference. If a landscape is imported as mesh (and not landscape), a landscape material is not necessary and the next step would be the import of satellite data.
