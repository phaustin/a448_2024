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

```{code-cell} ipython3
import rioxarray
import rioxarray.merge as riomerge
import xarray
from pathlib import Path
import cartopy.crs as ccrs
import cartopy
from matplotlib import pyplot as plt
import pycrs
```

## open the images

The default nasa projection is a [custom sinusoidal grid](https://pro.arcgis.com/en/pro-app/3.1/help/mapping/properties/sinusoidal.htm#:~:text=Sinusoidal%20is%20a%20pseudocylindric%20projection,central%20meridian%20and%20equally%20spaced.)

```{code-cell} ipython3
images = (Path.home() / "repos/a448_2024/data").glob("*tif")
images = list(images)
image1, image2 = images
rio_image1 = rioxarray.open_rasterio(image1, mask_and_scale = True)
rio_image2 = rioxarray.open_rasterio(image2, mask_and_scale = True)
wkt_text = rio_image1.spatial_ref.crs_wkt
wkt_text
```

```{code-cell} ipython3
proj4_crs = pycrs.parse.from_esri_wkt(wkt_text).to_proj4()
proj4_crs
```

```{code-cell} ipython3
rio_image1.rio.transform()
```

```{code-cell} ipython3
rio_image2.rio.transform()
```

```{code-cell} ipython3
rio_image1.spatial_ref
```

## squeeze out the band dimension

```{code-cell} ipython3
rio_image1.shape, rio_image2.shape
```

[test](https://spatialreference.org/ref/epsg/3157)

```{code-cell} ipython3
image1 = rio_image1.squeeze()
image2 = rio_image2.squeeze()
```

## reproject onto UTM Zone 10N

Reference [EPSG:3157](https://spatialreference.org/ref/epsg/3157) 

```{code-cell} ipython3
code = "EPSG:3157"
image1 = image1.rio.reproject(code)
image2 = image2.rio.reproject(code)
image1.spatial_ref.crs_wkt
```

## Plot images with cartopy

```{code-cell} ipython3
code = "3157"
projection = ccrs.epsg(code)
geodetic = ccrs.Geodetic()
```

```{code-cell} ipython3
image1.rio.transform(), image2.rio.transform()
```

bounds are 

```{code-cell} ipython3
image1.rio.bounds(), image2.rio.bounds()
```

## Get the coordinates for the bounding box

The upper left coordinates are 49.35N and 123.26W, and the lower right coordinates are 49.00N and 122.50W.

```{code-cell} ipython3
ul_lat, ul_lon = (49.35, -123.26)
lr_lat, lr_lon = (49, -122.50)
#ul_lat, ul_lon = (50, -126)
#lr_lat, lr_lon = (40, -121)
ul_x, ul_y  = projection.transform_point(ul_lon,ul_lat,geodetic)
lr_x, lr_y = projection.transform_point(lr_lon, lr_lat, geodetic)
extent = (ul_x,lr_x,ul_y,lr_y)
extent         
```

```{code-cell} ipython3
kw_dict = dict(projection=projection)
fig, ax1 = plt.subplots(1,1,figsize=(10,8),subplot_kw = kw_dict)
image1.plot.imshow(ax=ax1,origin="upper")
ax1.set_extent(extent, crs = projection)
ax1.set_title('image 1');
```

```{code-cell} ipython3
fig, ax2 = plt.subplots(1,1,figsize=(10,8),subplot_kw = kw_dict)
image2.plot.imshow(ax=ax2)
ax2.set_extent(extent, crs = projection)
ax2.set_title('image 2');
```

## find ul and lr corners of image 1

Use the [affine transform](https://www.perrygeo.com/python-affine-transforms.html)

```{code-cell} ipython3
image1.shape, image2.shape
```

```{code-cell} ipython3
ul_col, ul_row = ~image1.rio.transform()*(ul_x,ul_y)
ul_col,ul_row = int(ul_col), int(ul_row)
ul_col,ul_row
```

```{code-cell} ipython3
lr_col, lr_row = ~image1.rio.transform()*(lr_x, lr_y)
print(lr_col,lr_row)
lr_col,lr_row = int(lr_col), int(lr_row)
lr_col, lr_row
```

```{code-cell} ipython3
image1 = image1[ul_row:lr_row,ul_col:lr_col]
image1.shape
```

```{code-cell} ipython3
fig, ax2 = plt.subplots(1,1,figsize=(10,8),subplot_kw = kw_dict)
image1.plot.imshow(ax=ax2)
ax2.set_extent(extent, crs = projection)
ax2.set_title('image 1');
```

## now do the same for image 2

```{code-cell} ipython3
ul_col, ul_row = ~image2.rio.transform()*(ul_x,ul_y)
ul_col,ul_row = int(ul_col), int(ul_row)
ul_col,ul_row
```

```{code-cell} ipython3
lr_col, lr_row = ~image2.rio.transform()*(lr_x, lr_y)
lr_col,lr_row = int(lr_col), int(lr_row)
lr_col, lr_row
```

```{code-cell} ipython3
image2 = image2[ul_row:lr_row,ul_col:lr_col]
image2.shape
```

```{code-cell} ipython3
fig, ax2 = plt.subplots(1,1,figsize=(10,8),subplot_kw = kw_dict)
image2.plot.imshow(ax=ax2)
ax2.set_extent(extent, crs = projection)
ax2.set_title('image 2');
```

```{code-cell} ipython3

```