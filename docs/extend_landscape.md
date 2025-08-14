# Extending Landscapes

The plugin provides the ability to extend a Landscape. This is not the recommended way, but it may be useful if vast Landscapes should be created iteratively. It is also particularily useful to create backdrops (distant meshes).

> :bulb:  __Good to know__: For the playable area, the recommended way is to import large areas at once.  :bulb:

Said that, it is in principle possible to extend an imported area with any other raster files. One can also use data from Mapbox by selecting an area in the DTM Import Options.

The size of a landscape tile will always be the size of the first imported landscape. The resolution however can be altered by uncheck `Resample to first tile`. Additional imports will often completly overlap prior imports. Not wanted tiles can be deleted without side-effects.
