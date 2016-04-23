
output:
  md_document:
    variant: markdown_github
---

[![Travis-CI Build Status](https://travis-ci.org/mdsumner/spbabel.svg?branch=master)](https://travis-ci.org/mdsumner/spbabel)

<!-- README.md is generated from README.Rmd. Please edit that file -->


## Installation

Spbabel can be installed directly from github: 


```r
devtools::install_github("mdsumner/spbabel")
```
# spbabel: bisque or bouillon?

Part of an effort towards dplyr/table-fication of Spatial classes and the spatial-zoo in R. Inspired by Eonfusion and Manifold and dplyr and Douglas Adams and spurned on by helpful twitter and ropensci discussions. 

Spatial data in the `sp` package have a formal definition  (extending class `Spatial`) that is modelled on shapefiles, and close at least in spirit to the [Simple Features definition](https://github.com/edzer/sfr). See [What is Spatial in R?](https://github.com/mdsumner/spbabel/wiki/What-is-Spatial-in-R) for more details. 

Turning these data into tables has been likened to making fish soup, which has a nice nod to the universal translator [babelfish](https://en.wikipedia.org/wiki/List_of_races_and_species_in_The_Hitchhiker%27s_Guide_to_the_Galaxy#Babel_fish). Soup may be thick and creamy as in the "twice cooked" *bisque* or clear as in the "boiled" *bouillon* and this analogy can be applied in many ways including whatever way you like. 
 
# How does spbabel allow sp to work with dplyr? 

There are several ways to do this. 

* **1) dplyr-Spatial**:  Write the dplyr verbs for the Spatial classes (runs into limits with sp not using actual data.frame. 
* **2) sptable**:  Use the table of coordinates with identifiers for object and branch, like fortify, with sptable() and sptable()<- workflow to fortify and modify in a more flexible way. 
* **3) geometry-column** Use a real data.frame but include the "Spatial* objects as a list column. This is somewhat like the use of WKT or WKB geometry in tables, but without the need for constant (de)-serialization. 
* **4) single-level nesting**: Nest the fortify table for each object in a single column. 
* **5) double-level nesting**: Nest the twice-fortified table for each object, and then in the object table for each branch. This is close to full normalization, but cannot work with shared vertices or other topology. 
* **6) normalized-nesting**: Nest the normalized tables Vertices, Objects and the topological links between them. 
* **7) normalized**: A collection of related tables, without nesting. 


The normalized approaches are in flux, though a non-nested approach is well fleshed out already in [gris](https://github.com/msdumnser/gris). 

This document aims to illustrate each of these approaches, much is still work in progress. 

# Why do this? 

I want these things: 

* flexibility in the number and type/s of attribute stored as "coordinates", x, y, lon, lat, z, time, temperature, etc.
* shared vertices
* ability to store points, lines and areas together, sharing topology where appropriate
* provide a flexible basis for conversion between other formats.
* flexibility and ease of use
* integration with database engines and other systems
* integration with D3 via htmlwidgets, with shiny, and with gggeom ggvis or similar
* data-flow with dplyr piping as the engine behind a D3 web interface

The ability to use [Manifold System](http://www.manifold.net/) seamlessly with R is a particular long-term goal, and this will be best done(TM) via dplyr "back-ending". 

## "But, I don't like pipes!"

Is nothing sacred? 

Please note that the "pipelining" aspect of `dplyr` is not the main motivation here, and it's not even important. That is just a syntactic sugar, all of this work can be done in the standard function "chaining" way that is common in R. It's the generalization, speed, database-back-end-ability and need for flexibility in what Spatial provides that is key here. 


# 1) Direct dplyr verbs for Spatial

Apply `dplyr` verbs to the attribute data of `sp` objects with dplyr verbs. 

See `?dplyr-Spatial'` for supported verbs. 


```r
data(quakes)
library(sp)
coordinates(quakes) <- ~long+lat
library(spbabel)
## plot a subset of locations by number of stations
quakes %>% dplyr::filter(mag <5.5 & mag > 4.5) %>% select(stations) %>% spplot
```

![plot of chunk unnamed-chunk-3](figure/README-unnamed-chunk-3-1.png)

We can use polygons and lines objects as well. 


```r
library(maptools)
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

![plot of chunk unnamed-chunk-4](figure/README-unnamed-chunk-4-1.png)

```r

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

The problem with this approach is that it is fundamentally limited by the fact that Spatial*DataFrame objects are not actual data frames, they are very powerful but every feature that allows them to behave like data frames is painstakingly defined and extending this to the new features of extended data frames is onerous, and even slightly contentious. 

This approach is limited to the simple verbs  `arrange`, `distinct`, `filter`, `mutate`, `rename`, `select`, `slice`, `transmute`, and `summarize`. Summarize is a little bit of a stretch since it cannot be used after a `group_by` and so it is limited to collapsing to a single Spatial object, with all sub-geometries in one without any consideration of topology or relationships. 

# 2) sptable: a round-trip-able extension to fortify

The `sptable` function decomposes a Spatial object to a single table structured as a row for every coordinate in all the sub-geometries, including duplicated coordinates that close polygonal rings, close lines and shared vertices between objects. 

Use the `sptable<-` replacement method to modify the underlying geometric attributes (here `x` and `y` is assumed no matter what coordinate system). 


```r
## standard dplyr on this S4 class
(oz <- filter(wrld_simpl, NAME == "Australia"))
#> class       : SpatialPolygonsDataFrame 
#> features    : 1 
#> extent      : 112.9511, 159.1019, -54.74973, -10.05167  (xmin, xmax, ymin, ymax)
#> coord. ref. :  +proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs +towgs84=0,0,0 
#> variables   : 12
#> Source: local data frame [1 x 12]
#> 
#>     FIPS   ISO2   ISO3    UN      NAME   AREA  POP2005 REGION SUBREGION
#>   (fctr) (fctr) (fctr) (int)    (fctr)  (int)    (dbl)  (int)     (int)
#> 1     AS     AU    AUS    36 Australia 768230 20310208      9        53
#> Variables not shown: LON (dbl), LAT (dbl), rnames (chr)


## this is ready for ggplot2, though at the expense of needing the object attributes in a separate object
oztab <- sptable(oz)

##  make a copy to modify
woz <- oz
## modify the geometry on this object without separating the vertices from the objects
sptable(woz) <- sptable(woz) %>% mutate(x_ = x_ - 5)

# plot to compare 
plot(oz, col = "grey")
plot(woz, add = TRUE)
```

![plot of chunk unnamed-chunk-5](figure/README-unnamed-chunk-5-1.png)

We can also restructure objects, by mutating the value of object to be the same as "branch" we get individual objects from each.




```r

pp <- sptable(wrld_simpl %>% filter(NAME == "Japan" | grepl("Korea", NAME)))
## explode (or "disaggregate"") objects to individual polygons
## here each branch becomes an object, and each object only has one branch
## (ignoring hole-vs-island status for now)
wone <- spFromTable(pp %>% mutate(object_ = branch_))
op <- par(mfrow = c(2, 1), mar = rep(0, 4))
plot(spFromTable(pp), col = grey(seq(0.3, 0.7, length = length(unique(pp$object)))))
plot(wone, col = sample(rainbow(nrow(wone))), border = NA)
```

![plot of chunk unnamed-chunk-6](figure/README-unnamed-chunk-6-1.png)

```r
par(op)
```

# 3) geometry-columns: store the Spatial* thing in the column

The "geometry-column" approach stores the complex object in a special column type, or *as some kind of an evaluation promise* to generate that special type from the underlying data structure. How this is actually done in other programs worth exploring, see (Geometry columns)[https://github.com/mdsumner/spbabel/wiki/Geometry-columns]. 

In `spbabel` we can store the `Spatial*` (without the DataFrame) geometry in a special column. 


```r
geocol <- sp_df(wrld_simpl)
geocol %>% dplyr::select(NAME, Spatial_)
#> Source: local data frame [246 x 2]
#> 
#>                   NAME     Spatial_
#>                 (fctr)       (SptP)
#> 1  Antigua and Barbuda  Polygons[2]
#> 2              Algeria  Polygons[1]
#> 3           Azerbaijan  Polygons[5]
#> 4              Albania  Polygons[1]
#> 5              Armenia  Polygons[4]
#> 6               Angola  Polygons[3]
#> 7       American Samoa  Polygons[5]
#> 8            Argentina  Polygons[6]
#> 9            Australia Polygons[97]
#> 10             Bahrain  Polygons[6]
#> ..                 ...          ...
```

Note that this approach can be easily applied with WKT or WKB, it's just a fair bit of extra work to do the type conversion when needed. Note that, R has a `raw` type that seamlessly handles WKB from many DBs, and the `wkb` package can read it, likewise with `rgeos::readWKT` and `rgeos::writeWKT`. (See this in action in [manifoldr](https://github.com/mdsumner/manifoldr).)

## Questions on sp_df


Can we use mutate with the as coercion? 


```r
spdf <- sp_df(wrld_simpl)
## this doesn't work
#spdf %>% mutate(Spatial_ = as(Spatial_, "SpatialLines"))
spdf$Spatial_ <- as(spdf$Spatial_, "SpatialLines")
```

How to apply this, as an extra class? Note that geom_polygon in gggeom does not survive filter since the class attribute is lost. 


# 4) single-level nesting: tidy fortify

sportify?

# 5) double-level nesting: even tidier fortify

nsp_df

# 6) normalized-nesting: one table of non-analogous tables

db_df

# 7) normalized: gris

gris


# Issues

* names are not permanent yet, consider using "_" append like in gggeom for "special" columns
* need semantics for one-to-many and many-to-one copy rules, i.e. exploding an object copies? all attributes to the new objects or perhaps applies a proportional rule - see Transfer Rules in Manifold
* some details are carried by attr()ibutes on the objects, like "crs" - these don't survive the journey through functions without using classes - ultimately a branched object should carry this information with it
* 


# Early approach

Create methods for the dplyr verbs: filter, mutate, arrange, select etc. 

This is part of an overall effort to normalize Spatial data in R, to create a system that can be stored in a database. 

Functions `sptable` and `spFromTable` create `tbl_df`s from Spatial classes and round trip them. This is modelled on `raster::geom rather` than `ggplot2::fortify`, but perhaps only since I use raster constantly and ggplot2 barely ever. 

Complicating factors are the rownames of sp and the requirement for them both on object IDs and data.frame rownames, and the sense in normalizing the geometry to the table of coordinates without copying attributes. 

(See `mdsumner/gris` for normalizing further to unique vertices and storing branches and objects and vertices on different tables. Ultimately this should all be integrated into one consistent approach.)


Thanks to @holstius for prompting a relook under the hood of dplyr for where this should go: https://gist.github.com/holstius/58818dc9bbb88968ec0b

This package `spbabel` and these packages should be subsumed partly in the overall scheme: 

https://github.com/mdsumner/dplyrodbc
https://github.com/mdsumner/manifoldr


