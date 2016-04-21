[![Travis-CI Build Status](https://travis-ci.org/mdsumner/spbabel.svg?branch=master)](https://travis-ci.org/mdsumner/spbabel)

<!-- README.md is generated from README.Rmd. Please edit that file -->
Installation
------------

Spbabel can be installed directly from github:

``` r
devtools::install_github("mdsumner/spbabel")
```

spbabel: bisque or bouillon?
============================

Part of an effort towards dplyr/table-fication of Spatial classes and the spatial-zoo in R. Inspired by Eonfusion and Manifold and dplyr and Douglas Adams and spurned on by helpful twitter and ropensci discussions.

Spatial data in the `sp` package have a formal definition (extending class `Spatial`) that is modelled on shapefiles, and close at least in spirit to the [Simple Features definition](https://github.com/edzer/sfr). See [What is Spatial in R?](https://github.com/mdsumner/spbabel/wiki/What-is-Spatial-in-R) for more details.

Turning these data into tables has been likened to making fish soup, which has a nice nod to the universal translator [babelfish](https://en.wikipedia.org/wiki/List_of_races_and_species_in_The_Hitchhiker%27s_Guide_to_the_Galaxy#Babel_fish). Soup may be thick and creamy as in the "twice cooked" *bisque* or clear as in the "boiled" *bouillon* and this analogy can be applied in many ways including whatever way you like.

How does spbabel allow sp to work with dplyr?
=============================================

There are several ways to do this.

-   **1) dplyr-Spatial**: Write the dplyr verbs for the Spatial classes (runs into limits with sp not using actual data.frame.
-   **2) sptable**: Use the table of coordinates with identifiers for object and branch, like fortify, with sptable() and sptable()&lt;- workflow to fortify and modify in a more flexible way.
-   **3) geometry-column** Use a real data.frame but include the "Spatial\* objects as a list column. This is somewhat like the use of WKT or WKB geometry in tables, but without the need for constant (de)-serialization.
-   **4) single-level nesting**: Nest the fortify table for each object in a single column.
-   **5) double-level nesting**: Nest the twice-fortified table for each object, and then in the object table for each part. This is close to full normalization, but cannot work with shared vertices or other topology.
-   **6) normalized-nesting**: Nest the normalized tables Vertices, Objects and the topological links between them.
-   **7) normalized**: A collection of related tables, without nesting.

The normalized approaches are in flux, though a non-nested approach is well fleshed out already in [gris](https://github.com/msdumnser/gris).

This document aims to illustrate each of these approaches, much is still work in progress.

Why do this?
============

I want these things:

-   flexibility in the number and type/s of attribute stored as "coordinates", x, y, lon, lat, z, time, temperature, etc.
-   shared vertices
-   ability to store points, lines and areas together, sharing topology where appropriate
-   provide a flexible basis for conversion between other formats.
-   flexibility and ease of use
-   integration with database engines and other systems
-   integration with D3 via htmlwidgets, with shiny, and with gggeom ggvis or similar
-   data-flow with dplyr piping as the engine behind a D3 web interface

The ability to use [Manifold System](http://www.manifold.net/) seamlessly with R is a particular long-term goal, and this will be best done(TM) via dplyr "back-ending".

But I don't like pipes!
-----------------------

Please note that the "pipelining" aspect of `dplyr` is not the main motivation here, and it's not even important. That is just a syntactic sugar, all of this work can be done in the standard function "chaining" way that is common in R. It's the generalization, speed, database-back-end-ability and need for flexibility in what Spatial provides that is key here.

1) Direct dplyr verbs for Spatial
=================================

Apply `dplyr` verbs to the attribute data of `sp` objects with dplyr verbs.

See `?dplyr-Spatial'` for supported verbs.

``` r
data(quakes)
library(sp)
coordinates(quakes) <- ~long+lat
library(spbabel)
## plot a subset of locations by number of stations
quakes %>% dplyr::filter(mag <5.5 & mag > 4.5) %>% select(stations) %>% spplot
```

![](figure/README-unnamed-chunk-3-1.png)<!-- -->

We can use polygons and lines objects as well.

``` r
library(maptools)
#> Checking rgeos availability: TRUE
data(wrld_simpl)
## put the centre-of-mass centroid on wrld_simpl as an attribute and filter/select
worldcorner <- wrld_simpl %>% 
  mutate(lon = coordinates(wrld_simpl)[,1], lat = coordinates(wrld_simpl)[,2]) %>% 
  filter(lat < -20, lon > 60) %>% 
  select(NAME)

## demonstrate that we have a faithful subset of the original object
plot(worldcorner, asp = "")
text(coordinates(worldcorner), label = worldcorner$NAME, cex = 0.6)
```

![](figure/README-unnamed-chunk-4-1.png)<!-- -->

``` r

worldcorner
#> class       : SpatialPolygonsDataFrame 
#> features    : 6 
#> extent      : -178.6131, 179.0769, -54.74973, -10.05167  (xmin, xmax, ymin, ymax)
#> coord. ref. :  +proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs +towgs84=0,0,0 
#> variables   : 1
#> Source: local data frame [6 x 1]
#> 
#>                                  NAME
#>                                (fctr)
#> 1                           Australia
#> 2                       New Caledonia
#> 3                      Norfolk Island
#> 4 French Southern and Antarctic Lands
#> 5   Heard Island and McDonald Islands
#> 6                         New Zealand

## we can chain together standard operations as well as dplyr specific ones
wrld_simpl %>% as("SpatialLinesDataFrame") %>% summarise(big = max(AREA))
#> class       : SpatialLinesDataFrame 
#> features    : 1 
#> extent      : -180, 180, -90, 83.57027  (xmin, xmax, ymin, ymax)
#> coord. ref. : +proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs +towgs84=0,0,0 
#> variables   : 1
#> Source: local data frame [1 x 1]
#> 
#>       big
#>     (int)
#> 1 1638094
```

The problem with this approach is that it is fundamentally limited by the fact that Spatial\*DataFrame objects are not actual data frames, they are very powerful but every feature that allows them to behave like data frames is painstakingly defined and extending this to the new features of extended data frames is onerous, and even slightly contentious.

This approach is limited to the simple verbs `arrange`, `distinct`, `filter`, `mutate`, `rename`, `select`, `slice`, `transmute`, and `summarize`. Summarize is a little bit of a stretch since it cannot be used after a `group_by` and so it is limited to collapsing to a single Spatial object, with all sub-geometries in one without any consideration of topology or relationships.

2) sptable: a round-trip-able extension to fortify
==================================================

The `sptable` function decomposes a Spatial object to a single table structured as a row for every coordinate in all the sub-geometries, including duplicated coordinates that close polygonal rings, close lines and shared vertices between objects.

use the `sptable<-` replacement method to modify the underlying geometric attributes (here `x` and `y` is assumed no matter what coordinate system).

``` r
## standard dplyr on this S4 class
w2 <- filter(wrld_simpl, NAME == "Australia")
plot(w2, col = "grey")

## modify the geometry on this object
sptable(w2) <- sptable(w2) %>% mutate(x = x - 5)

plot(w2, add = TRUE)
```

![](figure/README-unnamed-chunk-5-1.png)<!-- -->

We can also restructure objects.

``` r
## explode (disaggregate) objects to individual polygons
## here each part becomes and object, and each object only has one part
#w3 <- spFromTable(sptable(w2)  %>% mutate(object = part, part = 1), crs = proj4string(w2))
```

3) geometry-columns: store the Spatial\* thing in the column
============================================================

sp\_df

4) single-level nesting: tidy fortify
=====================================

sportify?

5) double-level nesting: even tidier fortify
============================================

nsp\_df

6) normalized-nesting: one table of non-analogous tables
========================================================

db\_df

7) normalized: gris
===================

gris

Early approach
==============

Create methods for the dplyr verbs: filter, mutate, arrange, select etc.

This is part of an overall effort to normalize Spatial data in R, to create a system that can be stored in a database.

Functions `sptable` and `spFromTable` create `tbl_df`s from Spatial classes and round trip them. This is modelled on `raster::geom rather` than `ggplot2::fortify`, but perhaps only since I use raster constantly and ggplot2 barely ever.

Complicating factors are the rownames of sp and the requirement for them both on object IDs and data.frame rownames, and the sense in normalizing the geometry to the table of coordinates without copying attributes.

(See `mdsumner/gris` for normalizing further to unique vertices and storing parts and objects and vertices on different tables. Ultimately this should all be integrated into one consistent approach.)

Thanks to @holstius for prompting a relook under the hood of dplyr for where this should go: <https://gist.github.com/holstius/58818dc9bbb88968ec0b>

This package `spbabel` and these packages should be subsumed partly in the overall scheme:

<https://github.com/mdsumner/dplyrodbc> <https://github.com/mdsumner/manifoldr>
