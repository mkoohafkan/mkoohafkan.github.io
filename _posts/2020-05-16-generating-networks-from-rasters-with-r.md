---
layout:     post
title:      Generating networks from rasters with R
date:       2020-05-16 23:15:00
summary:    Making networks out of rasters turned out to be trivial with R.
categories: codemonkey r graph-theory
commentIssueId: 44
---

I've been doing a lot of work with networks and graph theory over the years,
both for my job at DWR and as a central component of my dissertation. 
I've been doing some stuff with
[spatial graphs](https://en.wikipedia.org/wiki/Spatial_network)
as part of an analysis of salmonid habitat in the 
[Russian River estuary](https://www.sonomawater.org/russian-river-estuary),
and this work involves analyses based on network representations of gridded
data. 

In a
[previous post]({% post_url 2020-01-30-generating-networks-from-polylines-with-arcgis %})
I showed you a very long and arduous process to convert polylines to
networks using ArcGIS. I initially turned to ArcGIS for converting
*rasters* to networks as well, focusing on using the
[Create Fishnet](https://pro.arcgis.com/en/pro-app/tool-reference/data-management/create-fishnet.htm)
and
[Polygon Neighbors](https://pro.arcgis.com/en/pro-app/tool-reference/analysis/polygon-neighbors.htm)
tools to convert rasters to shapefiles and generate the network edge list.
This turned out to be a very chalenging approach: generating the fishnet
took massive computing power, and my computer repeatedly failed to complete
the polygon neighbor analysis due to the large number of polygons. I needed
a different solution.

I eventually realized that the 
[`raster`](https://cran.r-project.org/package=raster) 
package for R had the tools I needed to generate a network directly
from the raster---no shapefiles needed! And it's only a few lines of
code. Generating the node and edge list of the network are just a single
function call each: the node list and their attributes can be generated
using the function `getValues`, while the `adjacent` function generates
a two-column matrix that is equivalent to a (bi)directional edge list.

I'll use the test raster provided by the package to demonstrate:

```r
library(raster)
r = raster(system.file("external/test.grd", package="raster"))
```

First we generate the node list using `getValues()`. We also get the location
info with a call to `coordinates()`:

```r
library(tidyverse)
nodes = tibble(value = getValues(r)) %>%
  bind_cols(as_tibble(coordinates(r)))
```

The edge list is generated with a call to `adjacent()`. We want the
adjacency information for every pixel, so we pass the complete set
of pixel indices for the `cells` argument. In this case only ask for
adjacency in the four cardinal directions, i.e. we don't check for
diagonal adjacency:

```r
edges = as_tibble(adjacent(r, cell = 1:ncell(r), directions = 4, sorted = TRUE))
```

Note that the node and edge lists include `nodata` pixels, which we
almost certainly don't care about. It doesn't noticeably impact the process
for this example, but for larger rasters it's probably smart to filter
out the `nodata` pixels beforehand and only retrieve adjacency info
for valid pixels. Here we simply filter out the `nodata` nodes after
creating the graph via the
[`tidygraph`](https://cran.r-project.org/package=tidygraph)
package:

```r
library(tidygraph)
g = tbl_graph(nodes, edges, TRUE) %N>%
  filter(!is.na(value))
```

And that's it! This is fine for smaller rasters, but you'll start
hitting problems with memory allocation as your rasters get larger.
I'm still working on how to handle that, but the fact that you can
choose to analyze adjacencies for a subset of pixels when using 
`adjacent()` means that it's easy to break up the analysis into chunks.
I'll write another post once I work out a good process.