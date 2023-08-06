---
layout: post
author:
- Richard Patterson
title: "Merge networks and simplify in R with the midlines package"
tags: [R, Rstats, network, cycle]
---
 

## Motivation

Where multiple versions of the same network are available and there are
small differences between them, it can be useful to create a merged
version against which each of the originals can be compared. An
alternative would be to consider one version as the gold standard and
compare others against that, but where there is no obvious
gold-standard it might be preferable to create to a merged version to
act as the gold standard.

The example used here is two sources of data on cycle lanes in an urban
area (although another might be different versions over time). They have
been recorded in different ways with different methods, leading to
variation in the exact position of the cycle lanes. One version of the
data is from [OpenStreetMap](https://openstreetmap.org/copyright) and
accessed for use in R with the
[osmextract](https://docs.ropensci.org/osmextract/) package. The second
version is provided by the local authority on the cycling
infrastructure in their area. There are clearly considerations about how cycle lanes
are defined and the comparability of these data, but here we put those
to one side and focus on creating the combined network.

This is still a work in progress and as such the example focuses on a
relatively small area of the network and aims to provide a proof of
concept.

## Setup

Packages are installed (if required) and loaded. The [midlines
package](https://github.com/RichardPatterson/midlines) is only available
from github, but the others are on CRAN and can be installed with the
install.packages() command.


``` {r}
#install.packages("sf")
#install.packages("dplyr")
#install.packages("nngeo")
#devtools::install_github("RichardPatterson/midlines")

# Load the required packages
library(sf)
library(dplyr)
library(nngeo)
library(midlines)
```

## The problem

In the plot below are two versions of the cycle lanes in a small urban
area. There are some bits of the network that are only present in one
version of the network and there are also some areas where both versions
agree that there is a cycle lane, but there are slight discrepancies in
its exact location.


``` {r}
plot(la_dat$geometry)
plot(osm_dat$geometry, add = TRUE, col = "RED")
```
![](/images/2023-08-06_network_merge/fig_1.png)

## A potential solution

The approach we will take is to draw a buffer around a combined dataset
of both versions of the data and then estimate the midline of the buffer
to approximate the merged network. The buffer distance is important as
we need to ensure that versions of the sames line create a single
buffer, without overlapping the buffers of nearby parts of the network.

``` {r}
# Merge cycle lane data
dat_both = rbind(osm_dat[,"geometry"],la_dat[,"geometry"])
buffer = st_buffer(dat_both, 7.5, endCapStyle = "SQUARE", joinStyle = "BEVEL" ) %>%
                              st_union() 
plot(buffer)
```

![](/images/2023-08-06_network_merge/fig_2.png)

The buffer is almost what we need, but some cleaning is required, for
example, there is a roundabout that is treated differently by the
different versions which has led to a 'hole' in the buffer. Luckily,
this is problem that other people have already solved for us. We can use
the st_remove_holes function from the
[nngeo](https://michaeldorman.github.io/nngeo/) package to specifiy the
size of holes to remove.

``` {r}
buffer = st_remove_holes(buffer, max_area = 250)
plot(buffer)
```

![](/images/2023-08-06_network_merge/fig_3.png)

## The midline

The next step is to estimate the midline of the buffer. I have put some
functions to help with this in a package available from github
([midlines package](https://github.com/RichardPatterson/midlines)). The
first step is to draw the midline, this uses a [Voronoi
tesselation](https://en.wikipedia.org/wiki/Voronoi_diagram) to estimate
the midline (heavily reliant on the st_voronoi function in the [sf
package](https://r-spatial.github.io/sf/)).

``` {r}
midlines_all = midlines::midlines_draw(buffer, dfMaxLength = 10)

plot(midlines_all$geometry)
```

![](/images/2023-08-06_network_merge/fig_4.png)

This results in a network that is broadly what we want, but the [Voronoi
tesselation](https://en.wikipedia.org/wiki/Voronoi_diagram) has resulted
in some unwanted short side branches. To remove these we can use the use
the midlines_clean function.

``` {r}
cleaned_lines = midlines::midlines_clean(midlines_all, n_removed = 4) %>%
  filter(removed_flag == 0)

plot(cleaned_lines$geometry)
```

![](/images/2023-08-06_network_merge/fig_5.png)

The midlines_clean function returns all the lines with some flagged for
removal, which is done by filtering the output. When exploring a new
network it can be useful to check that all unwanted lines are flagged
and all flagged lines are unwanted (the midlines package has additional
functions to help with this). With the unwanted side branches removed,
the result is the estimated merged and simplified network against which
each of the originals can be compared.

``` {r}
par(mfrow = c(1, 3))

plot(la_dat$geometry, main = "LA network")
plot(cleaned_lines$geometry, main = "Merged network")
plot(osm_dat$geometry, main = "OSM network")
```

![](/images/2023-08-06_network_merge/fig_6.png)

## Final thoughts

The parameters used here lead to the desired result on this small example.
Applying this approach to city-wide networks
or to networks from multiple cities is likely to yield additional
challenges and this is the next step. The approach outlined here
contains several trade-offs, the density of points in the buffer
influences the complexity of the estimated midline, which is likely to
more accurately represent the network, but may have more unwanted side
branches and so be more difficult to 'clean'. Cleaning the midlines
cannot always distinguish between the unwanted side branches and the
ends of lines in the desired network. The midlines package offers some
functions (and additional arguments to the functions used) to try and
address this, but the best combination is likely to be context
dependent. Additional description of ways this can be explored are found
in the [midlines package
readme](https://github.com/RichardPatterson/midlines).

## Attribution

The approach proposed here was inspired by Ferster et al[^1] and the
implementation of midline estimation was heavily influence by the very
comprehensive R package [`cmgo`](https://github.com/AntoniusGolly/cmgo)
which focuses on estimating river midlines and the Python library
[centerline](https://centerline.readthedocs.io/en/latest/). If my work
is useful for your work, I would appreciate it if you could cite the
midlines package.

[^1]: Ferster, C., Fischer, J., Manaugh, K., Nelson, T., Winters, M. 2020. Using OpenStreetMap to inventory bicycle infrastructure: A comparison with open data from cities. International Journal of Sustainable Transportation 14, 64--73. <https://doi.org/10.1080/15568318.2018.1519746>
