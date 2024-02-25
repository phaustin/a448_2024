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

## Accessing MODIS temperature data with the Planetary Computer STAC API

The planetary computer hosts three temperature-related MODIS 6.1 products:

- Land Surface Temperature/Emissivity Daily (11A1)
- Land Surface Temperature/Emissivity 8-Day (11A2)
- Land Surface Temperature/3-Band Emissivity 8-Day (21A2)

For more information about the products themselves, check out the User Guides at the [bottom of this document](#user-guides).

+++

### Environment setup

This notebook works with or without an API key, but you will be given more permissive access to the data with an API key.
The Planetary Computer Hub is pre-configured to use your API key.

```{code-cell} ipython3
---
vscode:
  languageId: python
---
import odc.stac
import planetary_computer
import pystac_client
import rich.table
```

### Data access

The datasets hosted by the Planetary Computer are available from [Azure Blob Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/). We'll use [pystac-client](https://pystac-client.readthedocs.io/) to search the Planetary Computer's [STAC API](https://planetarycomputer.microsoft.com/api/stac/v1/docs) for the subset of the data that we care about, and then we'll load the data directly from Azure Blob Storage. We'll specify a `modifier` so that we can access the data stored in the Planetary Computer's private Blob Storage Containers. See [Reading from the STAC API](https://planetarycomputer.microsoft.com/docs/quickstarts/reading-stac/) and [Using tokens for data access](https://planetarycomputer.microsoft.com/docs/concepts/sas/) for more.

```{code-cell} ipython3
catalog = pystac_client.Client.open(
    "https://planetarycomputer.microsoft.com/api/stac/v1",
    modifier=planetary_computer.sign_inplace,
)
```

### Query for available data

MODIS is a global dataset with a variety of products available within each larger category (vegetation, snow, fire, temperature, and reflectance). The [MODIS group](https://planetarycomputer.microsoft.com/dataset/group/modis) contains a complete listing of available collections. Each collection's format follows`modis-{product}-061`, where `product` is the MODIS product id. The `-061` suffix indicates that all of the MODIS collections are part of the [MODIS 6.1 update](https://atmosphere-imager.gsfc.nasa.gov/documentation/collection-61).


Let's access Land Surface Temperature/Emissivity Daily (11A1) data over Sacramento, CA in 2022 We'll get four images for the midseasonal months: March, June, September, and December. 
In the cell below we save the first granule for each month in the items dictionary

```{code-cell} ipython3
---
vscode:
  languageId: python
---
# Sacramento, CA
latitude = 38.6
longitude = -121.5
buffer = 2
bbox = [longitude - buffer, latitude - buffer, longitude + buffer, latitude + buffer]
year = "2022"
months = {
    "March": "03",
    "June": "06",
    "September": "09",
    "December": "12",
}
items = dict()
all_items = dict()

print("list. the number of images for each month\n")
# Fetch the collection of interest and print available items
for name, number in months.items():
    datetime = f"{year}-{number}"
    search = catalog.search(
        collections=["modis-11A1-061"],
        bbox=bbox,
        datetime=datetime,
    )
    all_items[name] = search.get_all_items()
    print(f"Available granules: {name}: {len(all_items[name])}")
    items[name] = all_items[name][0]

print("\ncloud fraction for first image\n")
for key, value in items.items():
    print(f"{key=}: {value.properties['eo:cloud_cover']=}")
```

### Available assets

Each item has several available assets, including the original HDF file and a Cloud-optimized GeoTIFF of each subdataset.

Print all the assets for the March image

```{code-cell} ipython3
---
vscode:
  languageId: python
---
t = rich.table.Table("Key", "Title")
for key, asset in items["March"].assets.items():
    t.add_row(key, asset.title)
t
```

### Loading the data

+++

For this example, we'll visualize the temperature data over Boise, Idaho. Let's grab each fire mask cover COG and load them into an xarray using [odc-stac](https://github.com/opendatacube/odc-stac). The MODIS coordinate reference system is a [sinusoidal grid](https://modis-land.gsfc.nasa.gov/MODLAND_grid.html), which means that views in a naïve XY raster look skewed. For visualization purposes, we reproject to a [spherical Mercator projection](https://wiki.openstreetmap.org/wiki/EPSG:3857) for intuitive, north-up visualization.

+++

### Reproject on to spherical mercator

see https://epsg.io/3857

this will transform all 4 images at the same time

```{code-cell} ipython3
---
vscode:
  languageId: python
---
data = odc.stac.load(
    items.values(),
    crs="EPSG:3857",
    bands="LST_Day_1km",
    resolution=500,
    bbox=bbox,
)

raster = items["March"].assets["LST_Day_1km"].extra_fields["raster:bands"]
data = data["LST_Day_1km"] * raster[0]["scale"]
```

```{code-cell} ipython3
raster
```

### Displaying the data

Let's display the temperature for each month.  

```{code-cell} ipython3
print(F"{type(data)=}, {data.shape=}")
```

```{code-cell} ipython3
g = data.plot.imshow(cmap="magma", col="time", vmin=260, vmax=300, size=4)
datetimes = data.time.to_pandas().dt.strftime("%B")

for ax, datetime in zip(g.axs.flat, datetimes):
    ax.set_title(datetime)
```

```{code-cell} ipython3
june_image = data[1]
from matplotlib import pyplot as plt
fig, ax = plt.subplots(1,1,figsize=(10,10))
june_image.plot.imshow(ax=ax);
```

## Download the June geotif

Click on the link below to get the geotif in your downloads folder

```{code-cell} ipython3
items['June'].assets['LST_Day_1km'].href
```

### User guides

- MOD11: https://lpdaac.usgs.gov/documents/715/MOD11_User_Guide_V61.pdf
- MOD21: https://lpdaac.usgs.gov/documents/1398/MOD21_User_Guide_V61.pdf
