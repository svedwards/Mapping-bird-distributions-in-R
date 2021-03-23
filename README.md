## Mapping birds and bicycle routes

This script shows how I overlayed my cross-country bicycle trip with the distribution of some regionally common birds. My bicycle journey was recorded in a file type called .gpx. It's a type of XML file I think, so the first thing I did was get the relevant libraries and read it in using the R package XML. [This tutorial](https://rpubs.com/ials2un/gpx1) helped.
```{r setup, include=FALSE}
library(XML)
library(OpenStreetMap)
library(lubridate)
xmlroute <- htmlTreeParse(file = "/Users/scottedwards/OneDrive\ -\ Harvard\ University/Family/bike/Sections/Presentations/Scotts_X_country_bicycle_trip_2020_v2.gpx",error = function(...) {
}, useInternalNodes = T)
```

Initially I wanted to use ggmaps for my map templates and mapping but as far as I could tell the diversity of maps available through ggmaps is somewhat limited. Instead I used ggplot mapping functions. First I parsed the XML object into elevations, times and coordinates:

```{r}
elevations <- as.numeric(xpathSApply(xmlroute, path = "//trkpt/ele", xmlValue))
times <- xpathSApply(xmlroute, path = "//trkpt/time", xmlValue)
coords <- xpathSApply(xmlroute, path = "//trkpt", xmlAttrs)
```

Then I cleaned up the files so that I end up with a simple data frame called "geodf":

```{r}
lats <- as.numeric(coords["lat",])
lons <- as.numeric(coords["lon",])
geodf <- data.frame(lat = lats, lon = lons, ele = elevations, time = times)
rm(list=c("elevations", "lats", "lons", "xmlroute", "times", "coords"))
```

If you want to include the times, you can make the format readable by R, although we won't use them further:

```{r}
geodf$time <- strptime(geodf$time, format = "%Y-%m-%dT%H:%M:%OS")
```

## Quick plot of the route
To  get a quick view of the route (as a plot):

```{r}
pdf(file = "route_plot.pdf",
    width = 16, # The width of the plot in inches
    height = 9) # The height of the plot in inches
plot(rev(geodf$lon), rev(geodf$lat), type = "l", lwd = 3, bty = "n", ylab = "Latitude", xlab = "Longitude")
dev.off()
```

## Getting the bird records from GBIF
Now let's get the map templates using ggplot, the bird distributions using rgbif, and generally get set up to plot. We'll read in 10000 records from gbif, each of which has coordinates, between the years 2006 and 2016. Additionally, we'll pick only records between the 6th and 8th months (June-August) to ensure breeding records.

```{r}
library(tidyr)
library(rgbif)
library("dplyr")
US_code <- isocodes[grep("United States", isocodes$name), "code"]
occur<-occ_data(scientificName = "Bartramia longicauda", country = US_code[2], hasCoordinate = TRUE, limit=10000, year = '2006,2016', month = '6,8')#
```

The occ_data function allows you to download just the lat, long and a few other important parameters, rather than all the gory details.
I found it then useful to isolate the data portion of the "occur" object:

```{r}
batramia<-occur$data
```

## Making the background map
Then I downloaded the template map. I wanted a simple black and white map with the boundaries of the states, so I used the borders function in ggplot2:

```{r}
library(ggplot2)
map_us <- borders(database = "state", colour = "gray50")
```

The lat and long data of each record in "batramia" is in the columns decimalLongitude and decimalLatitude. I wanted to get rid of records from Alaska so I filtered out the US records whose latitude was greater than 52:

```{r}
batramia<-filter(batramia,decimalLatitude < 52)
```

## Options for plotting
Then I use ggplot to plot the map and then lay the (now) 9932 bird records on it, using a transparency of 1/100 so you can see overlaps. I add the "blank background theme to get rid of the gray tiled background:
```{r}
ggplot() + map_us + geom_point(data = batramia,aes(x = decimalLongitude,y = decimalLatitude),alpha = 1/100,size = 5) + theme(panel.background = element_blank())
ggsave(file="Upland_sandpiper_gbif_plot_with_points.pdf", width=16, height=9)
```

You can also make this a gradient plot, although I find that, for this species, this interpolates some of the range gaps in a weird way. (I also find it looks less garbled in the regular R studio):

```{r}
ggplot(data = batramia,aes(x = decimalLongitude,y = decimalLatitude)) + map_us + stat_density_2d(aes(fill = ..level..),geom = "polygon", alpha = 0.4) + theme(panel.background = element_blank())
ggsave(file="Upland_sandpiper_gbif_plot_distributions.pdf", width=16, height=9)
```
## Putting it all together
I have no idea why the axis labels are now different! Probably something I did earlier.We can then overlay the bike route as a geom_line, adding another step to the ggplot. I think we plot the reverse of the lat and long so that the values are all positive. See [this web site](https://rpubs.com/ials2un/gpx1),which I used as s template.

```{r}
ggplot() + map_us + geom_point(data = batramia,aes(x = decimalLongitude,y = decimalLatitude),alpha = 1/100,size = 5) + theme(panel.background = element_blank()) +
geom_line(data=geodf,aes(x=rev(lon), y=rev(lat)),size=1,color="blue")
ggsave(file="Upland_sandpiper_gbif_plot_with_route.pdf", width=16, height=9)
```

## Other useful options
You can also plot subsets of the map by choosing particular states, filtering the bird records and bike route by lat and longs. I did this for the California Scrub Jay, which only occurred in Washington and Oregon on my route. It would be nice if there were a list of regions that could be accessed, but I couldn't find an easy one. Check the ggplot map functions:

dim(jays)
```{r}
jays<-occ_data(scientificName = "Aphelocoma californica", country = US_code[2], hasCoordinate = TRUE, limit=10000, year = '2006,2016', month = '6,8')
jays<-jays$data
jays<-filter(jays,between(decimalLatitude,42.00353622,49.00508118))
map_nw <- borders(database = "state", regions = c("washington","oregon"), colour = "gray50")
geonw<-filter(geodf,lon < -116.46513367)
ggplot(data = jays,aes(x = decimalLongitude,y = decimalLatitude)) + map_nw + stat_density_2d(aes(fill = ..level..),geom = "polygon", alpha = 0.4) + theme(panel.background = element_blank()) + 
  geom_line(data=geonw,aes(x=rev(lon), y=rev(lat)),size=1,color="blue")
ggsave(file="Scrub_jay_nw_plot_with_route.pdf", width=16, height=9)
```
