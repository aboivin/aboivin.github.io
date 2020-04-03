---
layout: post
title:  "Polygon targeting on facebook ads"
description: Give more precision to your facebook location based targeting
date:   2020-03-22 14:03:36 +0530
categories: facebook ads shapely kmeans GeoJSON
---

Facebook ads let you define your adset targeting based on circle with a minimum radius of one kilometer. Defining them manually can be tedious, that's why Facebook offers a lot of [APIs](https://developers.facebook.com/docs/marketing-apis/) to automate the tasks.

![Facebook location based targeting](/assets/facebook-targeting.png)

At teemo our targeting needs precision and each store may have catchment areas with a clean cut.
This post introduce a way to build polygon-like shapes using circles.


Geographical datastructures (Points, Line string or polygons) are generally encoded using one of the common GIS (Geographic information system) file format. The polygon in the following example will be represented using the [GeoJson](https://geojson.org/) format, a simple format based on JSON, widely used by open source GIS libraries.

Our approach
- Display GeoJSON format
- Sample points inside the polygon
- Use a k-means to cluster the points inside the polygon

## Display GeoJSON format

We are going to using python and *Jupyter lab* [geojson-extension](https://github.com/jupyterlab/jupyter-renderers/tree/master/packages/geojson-extension) support backed by a *leaflet* map. [troyes.geojson](/assets/troyes.geojson)

```python
from IPython.display import GeoJSON
import json

jsonPolygon = json.load(open('./troyes.geojson'))
GeoJSON(jsonPolygon)
```

![Troyes](/assets/troyes-kmeans-1-before.png)

## Sample points inside the polygon

With the *Shapely* python library we can find the bounding rectangle of or polygon `zone.bounds`. This rectangle can be sampled with points inside and outside the polygon. [Ray casting algorithm](https://en.wikipedia.org/wiki/Point_in_polygon) implemented by *Shapely* `zone.contains` method will help us to filter points inside the polygon.

```python
import numpy as np
from shapely.geometry import shape, Point

zone = shape(jsonPolygon)
min_x, min_y, max_x, max_y = zone.bounds

density = 20
x = np.linspace(min_x, max_x, density)
y = np.linspace(min_y, max_y, density)

sample_points = [Point(xi, yi) for xi in x for yi in y]
points_inside = [[p.x, p.y] for p in sample_points if zone.contains(p)]
```

## Use a k-means to cluster the points inside the polygon

Using sklearn k-means we can compute N clusters of points. N can be estimated as the number of 1km radius circles requiered to cover the zone area, ie. ` overlap coefficient * zone area / circle area`. The circles and the zone can be displayed overlayed.

```python
from sklearn.cluster import KMeans
from math import ceil, sqrt, pi
from shapely.geometry import mapping

# facebook circle of 1km radius 
facebook_circle_area = 0.0003822732994255957
overlap_coeff = 1.3
kmeans = KMeans(n_clusters=ceil(overlap_coeff * zone.area / facebook_circle_area), random_state=0).fit(points_inside)
centroids = kmeans.cluster_centers_

circles = [Point(p).buffer(sqrt(facebook_circle_area/pi)) for p in centroids]
generated_zone = [mapping(c) for c in circles]
GeoJSON(generated_zone + [jsonPolygon])
```
![dipslay](/assets/troyes-kmeans-after2.png)

The quality of the representation can be estimated with an IoU (Intersection over Union) metric, widely used in object detection field. The closer to 1 it is the better it fits. It could help further optimizations.

```
from shapely.ops import cascaded_union
circles_union = cascaded_union(circles)
IOU = (circles_union.intersection(zone)).area / cascaded_union([circles_union, zone]).area
IOU
```
`0.8630759942431909`

Here is a picture of the result `centroids` inputed in the facebook adset marketing API.

![result](/assets/troyes-kmeans-after-fb.png)

