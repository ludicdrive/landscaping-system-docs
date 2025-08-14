# Landcover

Applying a Landscape Material and generating __weightmaps__ on the fly is easy. In this example Shapefiles are used. Other formats are also supported.

> :bulb: __Good to know__: Unreal Engine paints the Landscape Material Layers according to the underlying resolution of the heightmap. The default is 1 vertex per meter (pixel/meter). The import resolution can be adjusted before import in the [DTM Options](heights.md?id=custom-raster-pixel-size) dialog. For more information on Unreal Engine paint layers for Landscape Materials, see [Unreal Engine Docs](https://dev.epicgames.com/documentation/en-us/unreal-engine/landscape-paint-mode-in-unreal-engine). :bulb:

## Get data from geofabrik.de

Landscaping System is designed to work with OpenStreetMap Data Extracts from Geofabrik. Other Shapefiles (also GeoJSON and GeoPackage) will also work with the right annotations (see [Add fclass to shapefile](landcover.md?id=add-fclass-to-shapefile)).  
[Download](https://download.geofabrik.de/) the Shapefiles from your country and extract the zip-file. For the example files please download the Shapefiles from [Austria](https://download.geofabrik.de/europe/austria.html) in Europe.

## Importing Vectordata

When clicking on the `Open` button next to Landscape Material Settings a new window pops up. Hit the `Select` button and choose `gis_osm_landuse_a_free_1.shp` or multiple files (see [Get Data](get-data.md?id=vector-data)) from the extracted files. If other vector data files than the Shapefiles from geofabrik.de are chosen, make sure they contain fclass attributes for the assignment later. It is possible to select several Shapefiles at once. It may take a few seconds until the file is read. The progress bar will pop up several times - once for each layer in the file. After that, choose the Landscape Material in the next step.

![Landscape Material and Weightmaps](_media/ue4_landscaping_weightmaps.jpg)  

## Landscape Material

Pick a Landscape Material from your content folder. Landscaping System comes with a debug Landscape Material included. For other more advanced Landscape Materials see [Unreal/Assetstore](https://unrealassetstore.com/), e.g. [Landscaping Auto Material](https://unrealassetstore.com/product/landscaping-auto-material/). The Landscape Material has to have layers.

## Material Layers

After the Landscape Material was selected, checkboxes for each layer of the material will show up. Here we can assign the layers of the Landscape Material to one or multiple landcover types (landuses). The `Landuse Types` are read from the flcass attribute of the shapes in the shapefile. One material layer can have one or more landuse types but not vice versa. Notice, that the column of the landcover type gets grayed out, if you assign it to a Material Layer. If a Material Layer has the same name as the landcover type, it will appear already selected. It is not mandatory, to select every landuse type nor Material Layer. Weightmaps will only be created for those landuses, which are assigned to a Material Layer. For everything else, the `Default Layer` is used.

> :bulb: __Good to know__: All Material Layers in use are generated as weightblended :bulb:

## Default Layer

The Default Layer is used everywhere on the landscape where no other landuse type is assigned.

## Noise Texture (optional)

Next to the layer name there are options to assign a noise texture, choose the noise textures color channel and set the minimum weight of the layer in the appropriate landuse type as well as the tiling of the noise texture. The noise texture will create variation blended with the default layer. There are two sample perlin noise textures in the content folder of the plugin.  

## Weightmaps generation

When finished with the mapping hit the `Replace` button. It will erase the current weightmaps and replace them with the new weightmaps.

> :bulb: __Good to know__: Depending on the extent of the landscape and the number of different landuse types, it will take several minutes (e.g. 100 km2) to hours (e.g. on a 1000 km2 landscape with a lot of different small areas) until all calculations are finished and you can see the result. :bulb:

## Add fclass to shapefile

If other shapefile used than this from geofabrik.de, it is possible to add an fclass attribute to the shapefile.

### Steps to add fclass attribute to shapefile

Add an attribute to an shapefile, using QGIS:  
1.) Load shapefile in QGIS  
2.) Right click it in Layers Tab and choose `Open Attribute Table`  
3.) Click `Edit` (pencil icon) and then `New Column` (Ctrl+W)  
4.) Fill out with  __Name=fclass, Type=string, Length= 64__  
5.) `Select All` (Ctrl+A)  
6.) Click `Multiediting Mode` (next to pencil icon) and type the value for the attribute (e.g. lake)  
7.) `Save` and close the dialog  
8.) Export the shapefile (right click on layer -> `Export...`)  
Then it will be compatible.

## Notes on runtime

The generation of weightmaps can take time. Especially if there are a lot of distinctive layers with small areas.
