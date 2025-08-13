# Procedural Content Generation Framework

Leverage Unreal Engines Procedural Content Generation (PCG) to accelarate procedural content creation.  
> Please enable `PCG` in `Edit -> Plugins` to use this feature.

## Automation

This feature helps to populate large worlds with procedural content. It spawns a PCG Volume on every tile/cell of the world with the selected PCG graph assigned.

## Data driven Procedural Content Generation (PCG) Graph

Of course the most powerful aspect of a PCG graph is to drive it with geospatial data. For example creating [weightmaps](landcover.md) with **Landscaping System** a PCG graph can sample the paint layer of the landscape. Furthermore it's possible to create all kinds of procedural content by implementing [custom logic](landscapingvectorinterface.md?id=custom-logic-on-vector-data) and spawn it through the vector data system.
