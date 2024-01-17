# parex
Simplify or pare a GeoJSON network simplification using raster image skeletonization and Voronoi polygons

Use image skeletonization or Voronoi polygons to simplify network, and output GeoPKG layers corresponding to the input, simplified and primal network. Where a primal network only contains straight line segments

The sample data set is of Queenstreet in Edinburgh kindly shared by Robin Lovelace

## Skeletonization
In an activated virtual environment, the following creates a simplified network by applying skeletonization to a buffered raster array
    
    (venv) $ ./skeletonize.py data/rnet_princes_street.geojson
   
## Voronoi
In an activated virtual environment, the following creates a simplified network by creating set of Voronoi polygons from points on the buffer
   
    (venv) $ ./voronoi.py data/rnet_princes_street.geojson

## Simple operation
The `run.sh` script setups a python virtual environment and executes the script against a data file in the `data` directory

    $ ./run.sh

The `run.sh` script optionally takes a filename and file-extension. To simplify a file, say `somewhere.geojson` and output to `GeoPKG` files `sk-thing.gpkg` and `vr-thing.gpkg`
    
    $ ./run.sh somewhere.geojon thing

## Notes
Both are the skeletonization and Voronoi approach are generic approaches, with the following known issues:

* This does not maintain a link between attributes and the simplified network
* This does not identify a subset of edges that need simplification
* The lines are a bit wobbly