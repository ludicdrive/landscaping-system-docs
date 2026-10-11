# Automation

Apart from its UI, Landscaping System can be driven from a script. Every operation that can be started from the `Landscaping System Tab` is reachable from Python, which also makes the plugin usable by AI agents through the Unreal Engine MCP server.

The scripted API reaches the same code as the UI. Nothing about the interactive workflow changes because of it, and the UI keeps working exactly as before.

> :bulb: __Good to know__: The scripted API is available in the editor only, on the Level that is currently open. It is not available in a packaged build. :bulb:

## Enabling scripting

Scripting is provided by the Unreal Engine `Python Editor Script Plugin` and the `Editor Scripting Utilities` plugin, so both need to be enabled in `Edit -> Plugins`. Landcaping System itself needs no extra setup for reading.

Writing needs one opt-in, see [Allow Automation Writes](settings.md?id=allow-automation-writes).

All entry points live on the `Landscaping System` editor subsystem:

```python
import unreal

landscaping = unreal.get_editor_subsystem(unreal.LandscapingManager)
```

### Naming: `b` prefixes are dropped

Unreal drops the `b` prefix from boolean properties and lowercases the rest, so the C++ `bSuccess` is `success` and `bValid` is `valid` in Python. Request fields are snake_case: `tile_indices`, not `TileIndices`.

Every request is a plain struct, so fields can be assigned directly or through `set_editor_property`. The two forms are equivalent.

## The safety model

Reading Level state never changes anything, so the read functions work without any opt-in.

Everything that modifies the Level is refused unless `Allow Automation Writes` is enabled. That covers importing files, downloading from Mapbox, creating Landscapes, creating Actors and Splines, and running PCG.

> ❗ __Important__: Nothing is ever saved automatically. Every operation leaves the Level dirty and says so in `needs_save`. Saving stays an explicit request from the caller. ❗

This matters because the interactive flow saves the Level on its own in a few places. When running from a script those saves are suppressed, so a scripted run cannot persist anything behind your back. A suppressed save is logged as a warning so it is visible that something happened.

## Reading Level state

Start here. `automation_get_level_info()` reports the state of the open Level:

```python
info = landscaping.automation_get_level_info()

print(info.number_of_tiles, info.projection, info.has_raster_data)
for tile in info.tiles:
    if tile.valid:
        print(tile.tile_index, tile.bounds, tile.number_of_spline_geometries)
```

`automation_get_tiles()` returns the same tile list if the Level state is not needed.

`tile_index` is the position in the list and the value to pass back as `tile_indices`. `valid` is the check the import path itself uses to decide whether a tile can be worked on, so filtering on it is what keeps a script from asking for a tile that cannot be imported.

Each tile reports the bounding box, location, scale, altitude range, the files it came from, and how many landuse shapes and Splines are loaded on it. The shape and Spline geometry itself is not exposed, only the counts.

`projection` is the EPSG code of the Level's coordinate reference system, and is `0` when the Level is not georeferenced yet, or when the CRS was accepted as a WKT that PROJ could not map back to an EPSG code. See [Coordinate reference systems](#coordinate-reference-systems) below.

> ❗ __Important__: An index that does not exist in the Level is logged as an error in the Output Log and skipped, rather than failing the request. `tiles_processed` is the reliable way to see what was actually reached. ❗

`selected_tiles` is what the Outliner currently has selected. A request that names no tiles is acted on exactly this set, so it is worth reading before relying on the fallback. An unattended run has no Outliner selection, which in practice means an empty `selected_tiles` and no tiles to work on.

## Importing files

```python
request = unreal.LandscapingImportRequest()
request.files = ["D:/data/roads.shp"]
request.kind = unreal.LandscapingAutomationImportKind.VECTOR
request.tile_indices = [0, 1, 2]

result = landscaping.automation_import_files(request)
print(result.success, result.error_message)
```

Every path is checked before the Level is touched, so a wrong path fails without having changed anything.

### Choosing the import kind

`kind` is explicit rather than derived from the file extension, because a Shapefile and a landuse Shapefile share the same extensions. The interactive UI distinguishes them by which button you pressed; a script has to say which it wants.

| Kind | Imports |
| :--- | :--- |
| `VECTOR` | Vector data, e.g. roads, rivers, railways |
| `LANDUSE` | Landuse data for weightmaps |
| `SATELLITE` | Satellite or NAPI imagery files |
| `DTM` | Heightdata, into the raster cache |

`layer_filter` optionally restricts a vector import to one named layer. Empty means all layers.

### Supported vector formats

`VECTOR` and `LANDUSE` accept Shapefiles, GeoJSON, GeoPackage and OSM files, filtered by `Allowed Vector File Types` in the settings. The three of Shapefile, GeoJSON and GeoPackage are equivalent here: given the same polygons they produce the same tiles and the same feature classes.

> :bulb: __Good to know__: GeoJSON is defined as EPSG:4326, so a GeoJSON import of projected data should be written with a reprojected CRS or expected to be read as geographic. GeoPackage keeps whatever CRS the layer was written with, which makes it the safest choice for projected data. :bulb:

### Selecting tiles

`tile_indices` are indices into the Level's tile list, as reported by `automation_get_tiles()`.

Leaving `tile_indices` empty falls back to the Outliner selection, which is what the interactive flow uses. Naming no tiles is not the same as naming none that exist - an empty list means "whatever is selected".

## Downloading from Mapbox

Landscaping System can also pull data straight from Mapbox, the same data the [Mapbox](mapbox.md?id=mapbox) buttons in the tab fetch, without a file on disk. A Mapbox API key has to be set in `Project Settings -> Plugins -> Landscaping Mapbox` first.

A Mapbox download is the one operation that does not finish before its call returns. It runs in the background, so it is split in two: `automation_fetch_from_mapbox()` starts it, and `automation_get_mapbox_result()` reports the outcome once it is done.

```python
import unreal

landscaping = unreal.get_editor_subsystem(unreal.LandscapingManager)

request = unreal.LandscapingMapboxRequest()
request.kind = unreal.LandscapingAutomationImportKind.VECTOR
request.zoom_level = 15
request.layer_filter = "water"
request.tile_indices = [0, 1, 2]

started = landscaping.automation_fetch_from_mapbox(request)
print(started.in_progress, started.tiles_processed)
```

The call returns as soon as the download has been handed off, with `in_progress` set and `tiles_processed` naming the tiles being fetched. A request that can be refused up front - no Mapbox module or API key, no tiles to fetch, a `DTM` without a material - comes back with `success` already `False` and an `error_message`, and nothing is started. Only one download runs at a time; a second fetch while one is in flight is refused the same way.

> ❗ __Important__: A download finishes on the editor's main thread, so the script has to return to the editor for it to progress. A blocking wait such as `while result.in_progress: pass` or `time.sleep` in a single script prevents it from ever completing. Start the fetch, then check the result in a later call, or poll from a Slate tick callback. ❗

Checking in a later call is the natural fit for an AI agent, which runs each tool call as its own script:

```python
result = landscaping.automation_get_mapbox_result()
if result.in_progress:
    print("still downloading")
else:
    print(result.success, result.error_message, result.vector_feature_classes)
```

For a single script that should see it through, poll from a post-tick callback so the editor keeps ticking while the download runs:

```python
handle = None

def poll(delta_seconds):
    global handle
    result = landscaping.automation_get_mapbox_result()
    if result.in_progress:
        return
    unreal.unregister_slate_post_tick_callback(handle)
    handle = None
    print(result.success, result.error_message, result.vector_feature_classes)

handle = unreal.register_slate_post_tick_callback(poll)
```

`automation_get_mapbox_result()` reports the same shape as every other operation. While the download runs `in_progress` is `True`; once it is done the stored result is returned with `in_progress` cleared. Asking before anything was started returns an error rather than an empty result.

### The request

| Field | Meaning |
| :--- | :--- |
| `kind` | What to download, see below |
| `zoom_level` | Mapbox zoom. `-1` keeps the plugin default for the kind: terrain `14`, vector and landuse `15`, satellite `16` |
| `layer_filter` | For `VECTOR` and `LANDUSE`, one or more Mapbox layer names, comma separated. Empty means all layers |
| `tile_indices` | The tiles to fetch and fill, as in a file import. Empty falls back to the Outliner selection |
| `material` | For `DTM`, the Landscape material. Optional, falls back to the Level's |
| `material_layers` | For `DTM`, the weightmap material layers |
| `raster_options` | For `SATELLITE` and `DTM`, the same options a [terrain request](satellite?id=satellite) takes |

### What each kind does

| Kind | Effect |
| :--- | :--- |
| `VECTOR` | Downloads vector tiles and imports them, exactly like a vector file import. Read `vector_feature_classes` afterwards |
| `LANDUSE` | Downloads landuse tiles for the weightmaps |
| `SATELLITE` | Downloads imagery and applies it to the Landscapes, the satellite button in script form |
| `DTM` | Downloads the heightmap and builds the Landscapes from it, so it covers both the DTM import and `create_terrains` in one call |

A `layer_filter` names Mapbox layers, not feature classes. The layers are matched exactly and case sensitively, and comma separates several. A river, for instance, is not a layer of its own: it is a feature with `class = 'river'` inside the `water` or `waterway` layer, so `water` is the layer to name. See [Mapbox](mapbox.md?id=mapbox) for what the tabs fetch.

> ❗ __Important__: A download of more than `Tile Download Warn Limit` tiles raises a confirmation dialog that a script cannot dismiss, so an unattended fetch stalls there. Keep scripted fetches under the limit or raise it in the [Mapbox settings](mapbox.md?id=tile-download-warn-limit). ❗

## Creating Landscapes

The Landscapes are built from what a DTM import left in the cache, so a DTM has to be imported first:

```python
# Import the heightdata first...
dtm = unreal.LandscapingImportRequest()
dtm.files = ["D:/data/dem.tif"]
dtm.kind = unreal.LandscapingAutomationImportKind.DTM
landscaping.automation_import_files(dtm)

# ...then build the Landscapes from it.
terrain = unreal.LandscapingTerrainRequest()
terrain.tile_indices = [0, 1, 2]
terrain.default_layer = "base"
landscaping.automation_create_terrains(terrain)
```

Calling this without raster data loaded returns an error rather than doing nothing.

> :bulb: __Good to know__: `material` is optional and falls back to the Level's Landscape material. Weightmap layers can be passed through `material_layers`. See [Landscape Material](landcover.md?id=landcover) for what the layers do. :bulb:

## Creating Actors, Splines and Blueprints

Requires vector data imported first, and at least one of a Spline Mesh, an Actor or Blueprint Class, or painting a material layer:

```python
actors = unreal.LandscapingBlueprintRequest()
actors.actor_or_blueprint_class = unreal.load_class(None, "/Game/BP_Tree")
actors.tile_indices = [0, 1]
actors.shape_f_class = "Circle"
actors.start_width = 100.0

landscaping.automation_create_blueprints(actors)
```

> :bulb: __Good to know__: When `paint_layer` names an existing Landscape paint layer, the interactive flow saves the Level once per tile before registering that layer. A scripted run skips those saves and warns in the Output Log instead. If the layer does not register, save the Level explicitly and check. :bulb:

See [Splines / Actors / Blueprints](props.md?id=props) for what the options do.

## Creating PCG volumes

Needs the `PCG` plugin enabled and a graph assigned:

```python
pcg = unreal.LandscapingPcgRequest()
pcg.graph_asset = unreal.load_asset("/Game/PCG/PCG_Forest")
pcg.tile_indices = [0, 1]
pcg.spawn_content = True

landscaping.automation_create_pcg(pcg)
```

See [PCG](vegetation.md?id=data-driven-procedural-content-generation-pcg-graph).

## The result

Every operation returns a `LandscapingOperationResult`:

| Field | Meaning |
| :--- | :--- |
| `success` | Whether the operation completed |
| `error_message` | Why it did not, empty on success |
| `status_message` | Progress information, if any |
| `in_progress` | A background Mapbox download is still running |
| `tiles_processed` | Tiles that actually received data |
| `vector_feature_classes` | Feature classes found, for vector imports |
| `needs_save` | The Level is dirty and was not saved |

### `tiles_processed` counts productive tiles

`tiles_processed` is deliberately not the list of tiles that were walked. It is the list of tiles that ended up with something in them, which is the only list that tells a caller whether the data actually landed:

| Kind | A tile is reported when |
| :--- | :--- |
| `VECTOR` | it has at least one spline geometry from this import |
| `LANDUSE` | it has at least one landuse shape from this import |
| `SATELLITE` | it received raster extents |
| `DTM` | it has a known raster extent, see below |

For `VECTOR` and `LANDUSE` the tile arrays are cleared before the import, so whatever is left afterwards belongs to this import and to nothing earlier.

A `DTM` is the exception worth knowing about. A heightmap describes the whole Level rather than dropping data on individual tiles, so `tiles_processed` reports the tiles that ended up carrying the raster extent. That is currently the single reference tile, not all 201 tiles the heightmap covers. Use it to confirm the DTM was read, not to count covered tiles. `create_terrains` afterwards reports the real tile count.

If an import completes but no tile ends up with data, the result is not a success. `success` becomes `False`, `tiles_processed` is emptied, and `error_message` explains the likely cause:

> Import finished without adding data to any tile. The file most likely does not overlap the Level, or its CRS could not be converted. See 'Projection Mode' in 'Project Settings -> Plugins -> Landscaping'.

This is the case that a file which misses the Level entirely would produce, and it is reported instead of being passed off as a successful no-op.

> ❗ __Important__: `success` means the operation completed, not that it changed anything. Always read `tiles_processed` to see what was actually touched. ❗

## Coordinate reference systems

A Level only works in a projected CRS whose unit is metre. Everything else has to be reprojected, which is what `Projection Mode` in [Settings](settings.md?id=projection-mode) controls.

The scripted API reads the resolved CRS back through `info.projection`:

- `projection` is the EPSG code of the Level CRS once one is known.
- It is `0` before any georeferenced data has been imported.
- It is also `0` when the Level CRS was accepted as a valid projected WKT that PROJ could not match back to an EPSG code. The import still works, because the WKT is carried through and used for the reprojection, but there is no code to report.

A projected WKT without a recognisable EPSG code is accepted rather than rejected. `MGI / Austria Lambert`, for instance, imports and builds Landscapes correctly and reports `0`.

An EPSG code is validated by asking PROJ whether it exists rather than by checking it against a numeric range, so a real code is never rejected for being small and an unknown one is not silently accepted.

A CRS that is geographic, or projected but not in metre, is refused as a Level CRS with an actionable error, pointing at UTM reprojection or a custom CRS. When that happens the error is surfaced through `error_message` rather than being buried in the Output Log.

> :bulb: __Good to know__: If shapes cannot be found inside the Landscapes although the file clearly overlaps, enable `Extended CRS Compatibility` in the settings. It falls back to a multi-step conversion when a single reprojection fails, at the cost of longer vector imports. :bulb:

## A complete example

Imports a DTM, builds the Landscapes, imports vector data in each of three equivalent formats, and places spline meshes on whatever actually received geometry:

```python
import unreal

DTM = "D:/data/dem.tif"
VECTORS = {
    "shapefile": "D:/data/water.shp",
    "geojson": "D:/data/water.geojson",
    "geopackage": "D:/data/water.gpkg",
}
KIND = unreal.LandscapingAutomationImportKind
EAL = unreal.EditorAssetLibrary

landscaping = unreal.get_editor_subsystem(unreal.LandscapingManager)


def check(label, result):
    """Every operation reports the same shape, so one helper covers them all."""
    tiles = list(result.tiles_processed)
    print("%-22s success=%-5s tiles=%-4d needs_save=%-5s %s"
          % (label, result.success, len(tiles), result.needs_save, result.error_message))
    if not result.success:
        raise RuntimeError("%s failed: %s" % (label, result.error_message))
    return tiles


# 1. Heightdata defines the Level extent and the CRS everything else is reprojected into.
request = unreal.LandscapingImportRequest()
request.files = [DTM]
request.kind = KIND.DTM
landscaping.automation_import_files(request)

# 2. Build the Landscapes from what the DTM left in the cache.
terrain = unreal.LandscapingTerrainRequest()
terrain.material = EAL.load_asset("/Engine/Tutorial/Landscape/TutorialAssets/Landscape_Material")
terrain.desired_max_tile_size = 1009
terrain.auto_calculate_landscape_size = True
check("create_terrains", landscaping.automation_create_terrains(terrain))

# 3. Import vector data and keep whichever format actually delivered geometry.
productive = []
for label, path in VECTORS.items():
    vector = unreal.LandscapingImportRequest()
    vector.files = [path]
    vector.kind = KIND.VECTOR
    tiles = check("vector_" + label, landscaping.automation_import_files(vector))
    if not productive:
        productive = tiles

if not productive:
    raise RuntimeError("no tile received geometry from any format")

# 4. Place spline meshes on the tiles that have splines, not on every tile.
actors = unreal.LandscapingBlueprintRequest()
actors.tile_indices = productive
actors.spline_mesh = EAL.load_asset("/Landscaping/Street/Props/RoadMesh")
actors.shape_f_class = "Circle"
actors.crop_to_bounds = True
actors.max_entities = 50
check("create_blueprints", landscaping.automation_create_blueprints(actors))

# 5. Nothing was saved. That is the caller's decision.
info = landscaping.automation_get_level_info()
print("projection=%s tiles=%d vector=%s raster=%s"
      % (info.projection, info.number_of_tiles, info.has_vector_data, info.has_raster_data))
```

Running it unattended needs the one opt-in, either in `Project Settings -> Plugins -> Landscaping` or in `Saved/Config/WindowsEditor/Landscaping.ini`:

```ini
[/Script/Landscaping.LandscapingSettings]
bAllowAutomationWrites=True
```

## Not covered yet

The following have no scripted entry point and still need the UI:

- Weightmap creation on its own, without creating Landscapes.
- Applying satellite imagery from an imported image file. The file import works, turning the images into decals does not. A Mapbox download does apply imagery, see above.
- The coordinate reference system check, which asks the user a question before edit layers are applied.

## AI agents

The same functions are reachable through the Unreal Engine MCP server, which exposes `BlueprintCallable` functions as tools. There is no Landscaping specific MCP plugin yet, the plugin is driven through the generic one.

> :bulb: __Good to know__: Because writes are opt-in and nothing is saved automatically, an agent can explore a Level with the read functions without being able to change it. :bulb: