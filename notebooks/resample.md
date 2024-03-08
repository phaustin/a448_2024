---
jupytext:
  formats: ipynb,md:myst
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.1
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# merging modis granules

This notebook takes geotiffs written with NASA's sinusoidal crs and resamples them onto a common
grid with a UTM zone 10 crs [epsg 3157 ](https://spatialreference.org/ref/epsg/3157)

We use [rioxarray](https://corteva.github.io/rioxarray/stable/) to get the crs information and [pyresample](https://pyresample.readthedocs.io/en/latest/concepts/index.html) to do the resampling.  The [pyproj](https://pyproj4.github.io/pyproj/stable/index.html) package is used to transform the lon/lat bounding box
into UTM coodinates for the area_def.

```{code-cell} ipython3
import rioxarray
import xarray
from pathlib import Path
import cartopy.crs as ccrs
import cartopy
from matplotlib import pyplot as plt
import numpy as np
import pyproj
import pyresample
from pyproj import Transformer, transform
from pyresample.geometry import AreaDefinition
```

## open the images using rioxarray

The default nasa projection is a [custom sinusoidal grid](https://pro.arcgis.com/en/pro-app/3.1/help/mapping/properties/sinusoidal.htm#:~:text=Sinusoidal%20is%20a%20pseudocylindric%20projection,central%20meridian%20and%20equally%20spaced.)

use rioxarray to get the coordinate reference system for the modis images

```{code-cell} ipython3
#images = (Path.home() / "repos/a448_2024/data").glob("*MYD*tif")
image_dir = Path.home() / ("Dropbox/phil_files/teaching/a448_2024/"
                         "propposals/hannah/MODIS_images/MODIS_Images")
year_dir = image_dir / "2021_MODIS_images"
images = list(year_dir.glob("*.tif"))
image1, image2 = images
rio_image1 = rioxarray.open_rasterio(image1, mask_and_scale = True)
rio_image2 = rioxarray.open_rasterio(image2, mask_and_scale = True)
wkt_text1 = rio_image1.spatial_ref.crs_wkt
```

Here is the sinusoidal crs in wkt format

```{code-cell} ipython3
rio_image1
```

```{code-cell} ipython3
wkt_text1
```

## Create three area_defs for pyresample

We need area_defs for the two modis tif files, and we need to create a third area_def that we are going to resample those two images into.





### make geodetic and utm zone 10N pyproj crs objects

We need to transform the lon/lat bounding box from geodetic coords (WGS84)  into utm coordinates (32610)

Start by creating two pyproj crs objects

```{code-cell} ipython3
crs_geodetic = pyproj.CRS("WGS84")
crs_zone10N = pyproj.CRS(32610)
crs_zone10N
```

```{code-cell} ipython3
crs_geodetic
```

### Get the region to resample into

We need the lower left and upper right corners of the region of interest in UTM coordinates
lat/lon is geodetic (epsg:4326)  and utm zone 10N is epsg:32610

**note that epsg:4326 is lat,lon  and epsg:32610 is x, y -- i.e. flipped**

```{code-cell} ipython3
ll_lat, ll_lon = (49, -123.26)
ur_lat, ur_lon = (49.35, -122.50)
transformer = Transformer.from_crs(4326, 32610)
ll_x, ll_y = transformer.transform(ll_lat,ll_lon)
ur_x, ur_y = transformer.transform(ur_lat,ur_lon)
print(f"{(ll_x,ur_x,ll_y,ur_y)=}")
area_extent = (ll_x,ur_x,ll_y,ur_y)
```

### get the pyresample area_defs from the images

Also get the crs (same for both images)

```{code-cell} ipython3
crs_nasa = pyproj.crs.CRS(wkt_text1)
rio1_def = pyresample.utils.rasterio.get_area_def_from_raster(str(images[0]),projection=crs_nasa)
rio2_def = pyresample.utils.rasterio.get_area_def_from_raster(str(images[1]),projection=crs_nasa)
rio1_def
```

```{code-cell} ipython3
rio2_def
```

### Construct the pyresample area_def for the bounding box

We need:

- area_id: ID of area
- description: Description
- proj_id: ID of projection (being deprecated)
- projection: Proj4 parameters as a dict or string
- width: Number of grid columns
- height: Number of grid rows
- area_extent: (lower_left_x, lower_left_y, upper_right_x, upper_right_y)


We want our area_extent to have these corners, need to project to utm zone 10 coords

+++

### Get the extent in UTM Zone10N coords

+++

### set the rest of the values

```{code-cell} ipython3
area_id = "vancouver"
projection = crs_zone10N.to_dict()
proj_id = "UTM Zone10"
description = "vancouver"
width = 60
height = 42
area_extent = ll_x, ll_y, ur_x, ur_y
```

### Make the destination area_def

```{code-cell} ipython3
vancouver_area_def = AreaDefinition(area_id, description, proj_id, projection,
                          width, height, area_extent)
```

```{code-cell} ipython3
vancouver_area_def
```

## resample both images into the vancouver area_def

```{code-cell} ipython3
from matplotlib import cm
from matplotlib.colors import Normalize
import copy
cmap = plt.get_cmap("plasma")
vmin=300
vmax = 330
the_norm=Normalize(vmin=vmin,vmax=vmax,clip=False)
over = 'w'
under='k'
missing='0.4'
cmap=copy.copy(cmap)
cmap.set_over(over)
cmap.set_under(under)
cmap.set_bad(missing)
```

```{code-cell} ipython3
data1 = rio_image1.data.ravel()
data2 = rio_image2.data.ravel()
out1 = pyresample.kd_tree.resample_nearest(rio1_def,data1, vancouver_area_def,radius_of_influence=500,fill_value=0)
out2 = pyresample.kd_tree.resample_nearest(rio2_def,data2, vancouver_area_def,radius_of_influence=500,fill_value=0)
cs = plt.imshow(out1,cmap=cmap,norm=the_norm)
fig = plt.gcf()
fig.colorbar(cs, extend="both");
```

```{code-cell} ipython3
cs = plt.imshow(out2,cmap=cmap,norm=the_norm)
fig = plt.gcf()
fig.colorbar(cs, extend="both");
```

## Create the merged image

```{code-cell} ipython3
merge = out1 + out2
hit = merge ==0
merge[hit]=np.nan
cs = plt.imshow(merge,cmap=cmap,norm=the_norm)
fig = plt.gcf()
fig.colorbar(cs, extend="both");
```

```{code-cell} ipython3
hit = merge > 500
merge[hit]=np.nan
```

```{code-cell} ipython3
cs = plt.imshow(merge,cmap=cmap,norm=the_norm)
fig = plt.gcf()
fig.colorbar(cs, extend="both");
```

```{code-cell} ipython3

```
