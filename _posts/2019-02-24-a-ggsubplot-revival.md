---
layout:     post
title:      A ggsubplot revival
date:       2019-02-24 15:15:00
summary:    There used to be  package for embedding subplots in a ggplot, but it's been deprecated for years. I figured out how to re-implement ggsubplot features within the tidyverse.
categories: codemonkey r ggplot2 ggsubplot embedded-plots
commentIssueId: 39
---

I'm settling into my new job at DWR, and one of my first tasks
has been some analysis of monitoring data for the
[Delta Smelt Resiliency Strategy](http://resources.ca.gov/delta-smelt-resiliency-strategy/).
We have long records of water quality data at a ton of stations,
and we've been doing some time series analysis to quantify
the effects of
[operating the salinity control gates in August](https://water.ca.gov/News/Blog/2018/Aug-18/Researchers-Test-New-Approach-to-Improve-Fish-Habitat-in-Suisun-Marsh)
on delta smelt habitat in
[Suisun Marsh](https://www.wildlife.ca.gov/Regions/3/Suisun-Marsh).
These data can be easily accessed from
[CDEC](http://cdec.water.ca.gov/); here I use the R package
[`CDECRetrieve`](https://cran.r-project.org/package=CDECRetrieve)
to pull continuously-monitored data on salinity (OK, technically
it's specific conductivity) at a few stations scattered across the
marsh. I produced hourly averages of the raw 15-minute data to
minimize issues with missing data and applied a 25-hour moving average
to filter out some of the noise in the salinity signal (including tides).

```r
library(CDECRetrieve)
library(purrr)
library(dplyr)
library(zoo)

# CDEC station codes to retrieve data for
stations = c('BDL', 'BLL', 'CSE', 'FLT', 'GOD', 'GZL', 'HON', 'HUN',
  'IBS', 'MAL', 'MRZ', 'MSL', 'NSL', 'PCT', 'RYC', 'TEA', 'VOL')

# get station locations
station.locations = map_dfr(stations, cdec_stations) %>%
  transmute(location_id = toupper(station_id), name,
    latitude, longitude)

# get station EC data
station.data = map_dfr(stations, cdec_query, sensor_num = 100L,
  dur_code = "E", start_date = '2016-09-25',
  end_date = '2016-11-05') %>%
  group_by(
    location_id,
    datetime = as.POSIXct(round(datetime, 'hours'))
  ) %>%
  summarize(mean = mean(parameter_value, na.rm = TRUE)) %>%
  mutate(windowmean = rollmean(mean, 25, 'center')) %>%
  ungroup()
```

I'm always interested in relationships between components of a
system, and I often find it helpful to simultaneously look at both the
spatial relationships between and temporal trends across the data.
When data is collected at discrete locations, one of the most 
straightforward ways to do this is simply plot the time series graphs 
on a map. This lets your eye quickly identify similar patterns in the 
time series as well as spatial gradients in the system.

Many years ago there was a package called
[`ggsubplot` ](https://github.com/garrettgman/ggsubplot) that could
be used exactly for this sort of thing, but it wasn't maintained
and was eventually removed from CRAN. I've fantasized about trying
to rebuild it myself, but never had the time or the motivation to
learn about the
[inner workings of `ggplot2`](https://ggplot2.tidyverse.org/articles/extending-ggplot2.html).
I was actually working on a completely unrelated problem---trying
to understand the difference between how
[`ggmap`](https://cran.r-project.org/package=ggmap) and
[`ggspatial`](https://cran.r-project.org/package=ggspatial) plot
basemaps---when I stumbled across the `ggplot2` function
[`annotation_custom`](https://ggplot2.tidyverse.org/reference/annotation_custom.html):

> This is a special geom intended for use as static annotations
that are the same in every panel. These annotations will not
affect scales (i.e. the x and y axes will not grow to cover the
range of the grob, and the grob will not be modified by any ggplot
settings or mappings).

Woah. The exciting word here is "grob"---because ggplot objects can 
themselves be turned into grobs
(see [`ggplotGrob`](https://ggplot2.tidyverse.org/reference/ggplotGrob.html).
Could I use this to drop a plot on, say, a map made with ggplot?

The first step is to make all the plots. This is actually really
easy to do with `purrr`, `tidyr`, and `tibble` support for
list-columns.

```r
library(tidyr)
library(ggplot2)

# make a separate ggplot for each station
station.plots = station.data %>%
  nest(-location_id) %>%
  mutate(plot = map2(data, location_id,
    ~ ggplot(.x) + ggtitle(.y) + theme_bw(base_size = 8) +
    aes(x = datetime, y = windowmean) + geom_line() +
    scale_x_datetime(NULL, breaks = as.POSIXct(c("2016-10-01",
      "2016-10-15", "2016-10-31")), date_labels = "%b %d",
      limits = as.POSIXct(c("2016-10-01", "2016-10-31"))) +
    scale_y_continuous("Spec. Conductivity",
      limits = c(4000, 30000))
    )
  )
```

I also need a spatial layer that defines where the plots will
be placed. I use the
[`sf`](https://cran.r-project.org/package=sf) package to
convert the station locations to spatial points:

```r
station.points = st_as_sf(station.locations, crs = 4326,
  coords = c("longitude", "latitude")) %>%
  st_transform(3857)
```

Next, I need to turn the plots into annotations. Again, this
is easy with `purrr`. I extract the station coordinates
from the spatial layer and join them to the plot data, and then
generate a separate annotation layer for each plot. Note that
I need to convert each plot to a grob via `ggplotGrob` and
explicitly define the bounding box (`xmin`/`xmax`/`ymin`/`ymax`)
of each annotation layer.

```r
station.annotations = station.points %>%
  bind_cols(as_tibble(st_coordinates(.))) %>%
  st_drop_geometry() %>%
  select(location_id, X, Y) %>%
  left_join(station.plots, by = "location_id") %>%
  mutate(annotation = pmap(list(X, Y, plot),
    ~ annotation_custom(ggplotGrob(..3),
      xmin = ..1 - 2000, xmax = ..1 + 2000,
      ymin = ..2 - 1000, ymax = ..2 + 1000))) %>%
  pull(annotation)
```

Finally, I build the final ggplot map. I use `ggspatial`
to obtain a basemap and simply add the list of annotations
to the plot. I don't actually want to plot a point at each
station location, so I pass `shape = NA` to `geom_sf` to
hide the points (although they would be covered up by the
plot annotations anyway).

```r
library(ggspatial)
ggplot(station.points) +
  xlim(c(-13598000, -13563500)) +
  annotation_map_tile(zoom = 13) +
  geom_sf(shape = NA) +
  station.annotations
```

![Inset time series of specific conductivity overlaying a map of Suisun Marsh](/images/salinitymaps.svg)

Not bad! You do have to be careful with how you define your
plots, because `annotate_custom` doesn't try to make any
adjustments to the plots when placing them on the map. My code 
above has a lot of formatting in my plot statement to reduce
the base font size, set consistent axis limits, and reduce the
number of breaks in the axes. There will inevitably be some iteration 
required to produce a map you are happy with, but it beats making the
maps by hand in GIS software. Also note that you're not restricted
to just placing one kind of subplot at a time, so you could make a
line plot here, a histogram there, etc. to create any combination of
subplots you want.

It can't be too hard to turn this process into an actual ggplot
geometry (`geom_subplot` perhaps?) that takes care of some of
the plot sizing and tweaking automatically, but I don't have
the motivation right now. Maybe someday I'll take a stab at
making `ggsubplot2`...
