---
layout: page
title: "Obtaining and cleaning species occurrence data in R"
author: "Jon Spoelhof"
permalink: /SpoelhofJ/

---



### Before we begin...

All of the following packages are necessary for this module. If you haven't  already, install them with ``install.packages``. In a Linux environment, it may be necessary to install the ``libgdal-dev`` library in the terminal before installing ``rgdal`` in R.

```{r}
#install.packages("dplyr")
library(dplyr)
#install.packages("raster")
library(raster)
#install.packages("rgdal")
library(rgdal)
#install.packages("maps")
library(maps)
#install.packages("spocc")
library(spocc)
#install.packages("taxize")
library(taxize)
```

Next, set a working directory.

```{r}
setwd("your_directory_file_path")
```

We need to create two functions. The first is a function to capitalize the first letter of a character string. This will come in handy for formatting species names later. The second is a function that cleans up the output of our taxonomic name resolution service and returns a simple vector of names.

```{r}
firstup <- function(x) {
  substr(x, 1, 1) <- toupper(substr(x, 1, 1))
  x
}

post_tnrs = function(name_query){
  accepted_species = name_query$acceptedname
  #fill in blanks with matched name or submitted name (if no match)
  blanks = which(accepted_species == "")
  accepted_species[blanks] = name_query$matchedname[blanks]
  blanks = which(accepted_species == "" | name_query$score <= 0.5)
  accepted_species[blanks] = name_query$submittedname[blanks]
  #restore submitted name order
  accepted_species = accepted_species[match(species, 
                                            name_query$submittedname)]
  #remove special character from hybrid species
  hybrids = grepl(" × ", accepted_species)
  accepted_species[!hybrids] = unlist(lapply(lapply(strsplit(accepted_species[!hybrids], 
                                                             split = " "), 
                                                    function(x) x[1:2]), 
                                             paste, 
                                             collapse = " "))
  accepted_species[hybrids] = unlist(lapply(lapply(strsplit(accepted_species[hybrids], 
                                                            split = " "), 
                                                   function(x) x[c(1,3)]), 
                                            paste, 
                                            collapse = " "))
  return(accepted_species)
}
```

## Species of interest

### Example: *Epidendrum*

I like orchids, particularly *Epidendrum* species. After looking around the internet for a few interesting species, I found these species names: *Epidendrum arachnoides*, *Epidendrum nocturnum*, and *Epidendrum ciliaris*

Let's imagine that we want to become international biocriminals and go smuggle some specimens of these species from their native range without a permit (or that we want to build niche models, whatever). To do that, we will need to use these names to extract occurrence data from a database like GBIF or iDigBio, which have aggregated plant occurrence and specimen data from collections around the world.

First, however, we have to make sure we have the right names, not old names or synonymous names. We can check our names against databases of accepted species names using a taxonomic name resolution service, or TNRS. We'll use is the ``tnrs`` function in the R package ``taxize``.

```{r}
species = c("Epidendrum arachnoides",
            "Epidendrum nocturnum",
            "Epidendrum ciliaris")

(name_query = tnrs(species,
                   source = "iPlant_TNRS"))
```

``tnrs`` gives you some pretty useful information. In addition to the names. The match score can be used to filter out cases where it can only match the genus name (score <= 50%). In our example the matches are of good quality (> 90%). The names we will use for database queries are the accepted names. Our function ``post_tnrs`` would replace any questionable accepted names with the matched name or submitted name (in that order), as well as restoring the input order.

```{r}
accepted_species = post_tnrs(name_query)
```

Now, let's compare these names to the input names.

```{r}
species; accepted_species
```

One of our species names was correct, one had a typo, and the other had a synonymous accepted name.

## Occurrence database queries

Most occurrence databases (GBIF, iDigBio, Bison, eBird, iNaturalist, etc) have functional R based APIs. The ``spocc`` package allows you to query multiple APIs at once through the ``occ`` function. We'll use it here. 

First, let's specify the sources we want to use. for our *Epidendrum* example, we'll use iDigBio and GBIF.

```{r}
sources = c("idigbio", "gbif")
```

Next, we'll format our query. ``occ`` has several options to filter the data that is returned. We'll limit our records per species to 1,000 and set ``has_coords = T``, so that we only retrieve records with lon/lat coordinates. We can also use ``gbifopts`` and ``idigbioopts`` to set options that are passed to the individual APIs (see the documentation for packages ``ridigbio`` and ``rgbif`` for more options), in this case, a specification that only records attached to a preserved specimen are returned. This will eliminate many unwanted records, including observations, which are typically less reliably identified than preserved specimens, and living specimens, which often come from unnatural habitats like zoos or botanical gardens.

```{r}
occurrences = occ(accepted_species,
                  from = sources,
                  limit = 1000,
                  has_coords = T,
                  idigbioopts = list(rq = list(basisofrecord = "preservedspecimen")),
                  gbifopts = list(basisOfRecord = "PRESERVED_SPECIMEN"))
                 
occurrences = fixnames(occurrences,
                         how = "query")
```

This query returns a large 'occdat' object with individual results from each database. ``fixnames`` simply ensures that the names in the output match the names we supplied in our query. ``spocc`` provides the ``occ2df`` function as a utility to merge query results from different databases into a single data frame.

```{r}
occdf = occ2df(occurrences)
```

While this function is very convenient, it eliminates much of the raw information that will be useful for cleaning the occurrence data later. It's a better idea to compile the raw data manually. Unfortunately, the ``occdat`` object structure does not make this very easy. It contains data frames for each source-species combination that we'll have to extract separately and combine.

First, we'll need to generate each potential combination of species and source with ``expand.grid``. Note the use of ``gsub`` to replace the space between the genus and specific epithet with an underscore.

```{r}
(source_species_comb = expand.grid(sources,
                                   gsub(" ",
                                        "_",
                                        accepted_species)))
```

Next, we'll use these combinations to assemble strings that will extract individual data frames from the ``occdat`` object.

```{r}
(source_species_string = paste("occurrences",
                               source_species_comb$Var1,
                               "data",
                               source_species_comb$Var2,
                               sep = "$"))
```

Finally, we can evaluate these strings within the ``bind_rows`` function to pull all of the data into one raw data set.

```{r}
raw = eval(parse(text = (paste("bind_rows(",
                               paste(source_species_string,
                                     collapse = ","),
                               ")"))))
```

There are a lot of columns in this data set, and most of them are useless. We can create a new data frame from the raw data set by specifying the columns we want to keep (in this case: name, longitude, latitude, date, country, locality, data provider, and unique key. Unfortunately GBIF and iDigBio occasionally use different column names for the same data types. In these cases, alternate columns will either contain data or NA based on the source, so we can simply use ``pmin`` to merge them within the ``data.frame`` function. We'll also round the lon/lat coordinates to resolve any floating-point number differences between identical records from different sources. This will make removing duplicate records easier later.

```{r}
occ_total = data.frame(name = raw$name,
                       longitude = round(raw$longitude,
                                         digits = 5),
                       latitude = round(raw$latitude,
                                        digits = 5),
                       date = pmin(raw$eventDate,
                                   raw$datecollected,
                                   na.rm = T),
                       country = raw$country,
                       locality = raw$locality,
                       prov = raw$prov,
                       key = pmin(raw$key,
                                  raw$uuid,
                                  na.rm = T),
                       stringsAsFactors = F)
```

Now, let's find out how many records we obtained for each species

```{r}
(init_species_table = table(occ_total$name))
```

We finally have a perfectly usable raw data set. Now, the hard part...

## Data cleaning

Occurrence data are notoriously messy, and there are a lot of steps to take before we can consider a data set to be 'clean'. The following steps are geared toward producing a reliable set of natural occurrences of the species of interest (e.g. for niche modeling). 

### Missing data

Our raw data set has many fields, of which quite a few are likely to have missing data. the fields which are absolutely essential are species name, date, longitude, and latitude. All records will possess a species name, but not all records will possess dates or valid lon/lat coordinates.

Removing missing dates is easy with ``is.na``:

```{r}
occ_total = occ_total[!is.na(occ_total$date),]
```

You may also want to restrict occurrence dates to a certain range. For example, the following code would remove any occurrences dated before Jan. 1, 2000.

```{r}
#occ_total = occ_total[occ_total$date > "2000-01-01",]
```

Although we specified ``has_coords = T`` in our ``occ`` query, some missing coordinate values will have been entered database as 0 lon / 0 lat. Given that this point is in the Atlantic Ocean south of Ghana, it's unlikely that any of our species of interest (all epiphytic plants) will naturally occur there. We can remove these records with simple subsetting.

```{r}
occ_total = occ_total[(occ_total$longitude != 0) & (occ_total$latitude != 0),]
```

### Duplicated records

R provides many ways to identify and remove duplicated data. Keep in mind that, for spatial data, part of this process will involve matching lon/lat coordinates as floating-point numbers. Using a function like ``duplicated`` is very simple, but, had we not rounded the lon/lat coordinates earlier, it would not remove many duplicate records because of residual floating-point differences between numbers from different sources. While a function like ``all.equal`` could be applied instead, the method presented here is much simpler and faster, particularly for large data sets.

```{r}
duplicates = duplicated(occ_total[,c("name", "longitude", "latitude")])

occ_total = occ_total[!duplicates,]
```

Note that this code will remove records taken from exactly the same location, but on different dates. This would be appropriate for a niche modeling approach (subsequent sampling would usually be redundant from a modeling perspective), but not necessarily appropriate for other uses where sampling history is importance.

### Questionable records

Our data set is mostly clean, but there will inevitably be a few records that are incorrect or dubious. The following steps may seem like a lot of work to remove a few points here and there, but they are a good way to remove outliers that could influence subsequent analyses.

First, we want to remove any records whose lon/lat coordinates don't match their stated country of origin. ``map.where`` from the ``maps`` package will return a vector of countries based on a set of input coordinates.

```{r}
countries = map.where(database = "world", occ_total$longitude, occ_total$latitude)

countries = gsub("\\:.*","",countries)
```

The ``gsub`` function was use here to remove the additional location names after country.

Now, we'll want to check the output from ``map.where`` against the vector of countries associated with our occurrence records, but there's a problem: the country names from iDigBio and GBIF are an inconsistent mess.

```{r}
unique(countries)
unique(occ_total$country)
```

So, we can identify the specific country names that don't match the names from ``map.where``...

```{r}
(bad_country_names = unique(occ_total$country)[!(unique(occ_total$country) %in% unique(countries))])
```

...and create an index to correct those names (note that Jamaica does not appear in the ``map.where`` output; this was a mismatch)...

```{r}
country_correction = data.frame(old = bad_country_names,
                                new = c("Brazil", "Brazil", "Bolivia",
                                        "Mexico", "USA", "Venezuela",
                                        NA, "USA", "Trinidad", NA),
                                stringsAsFactors = F)
```

and change those names with a ``for`` loop.

```{r}
for(i in 1:nrow(occ_total)){
  if(occ_total$country[i] %in% country_correction$old){
    occ_total$country[i] = country_correction$new[country_correction$old == occ_total$country[i]]
  }
}
```

Finally, we can filter out any records with mismatched countries and lon/lat coordinates (and those that are now ``NA``).

```{r}
occ_total = occ_total[countries == occ_total$country,]
occ_total = occ_total[!is.na(occ_total$country),]
```

Next, we'll want to filter out any records with mis-located records that don't fall on land. To do this, we need global maps of land and major lakes. We can download and import these maps into R with the following code.

```{r}
land_url = "www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_land.zip"
lakes_url = "www.naturalearthdata.com/http//www.naturalearthdata.com/download/10m/physical/ne_10m_lakes.zip"

download.file(land_url,
              "ne_10m_land.zip")

download.file(lakes_url,
              "ne_10m_lakes.zip")

unzip("ne_10m_land.zip",
      exdir = "./ne_10m_land")
unzip("ne_10m_lakes.zip",
      exdir = "./ne_10m_lakes")

land = readOGR("ne_10m_land")
lakes = readOGR("ne_10m_lakes")
```

We can check the lon/lat coordinates of our occurrences against these maps to make sure the none of the records came from dubious aquatic locations (if you were searching for shark occurrences, you would do the opposite, obviously).

First, we need to make convert our coordinates into a ``SpatialPoints`` object with the same projection as our maps...

```{r}
global_occs = data.frame(x = occ_total$longitude, y = occ_total$latitude)
coordinates(global_occs) = ~ x + y
proj4string(global_occs) = CRS("+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 +towgs84=0,0,0")
```

...then use ``over`` to check each point against the maps. In this case, we want to keep any points that are over land and not over lakes. We can filter out points that don't meet these criteria easily because over returns ``NA`` for any point not over the specified feature of the map.

```{r}
over_land = over(global_occs, land)
over_lakes = over(global_occs, lakes)

occ_total = occ_total[(!is.na(over_land[,1])) & (is.na(over_lakes[,1])),]
```

This method of spatial filtering is incredibly useful. In addition to checking points against maps of land, you can check points against maps of specific countries or municipal boundaries, soil maps, maps of research plots, maps of previously glaciated areas, etc.

We've done just about everything we can without looking at the points visually to identify obvious outliers. Lets plot our points on a map to check.

```{r}
par(mar = c(0,0,0,0))
plot(land)

legend("topright",
       unique(occ_total$name),
       pch = 16,
       col = c("dark red", "navy", "lime green")[unique(factor(occ_total$name))])

points(occ_total$longitude,
       occ_total$latitude,
       pch=16,
       cex = 0.5,
       col = c("dark red", "navy", "lime green")[factor(occ_total$name)])
```
![global_map](figures/Jon_image.png)

Well, there are some weird points, mostly in the USA. Note the points in Hawaii, New England, and central USA. Those points are definitely outside of the normal range of these orchids, but they weren't caught by any of our filtering steps this far. Let's look at the locality data to find out what's going on. We can isolate the records of interest via sorting by latitude and filtering by country.

```{r}
latsort = occ_total[order(occ_total$latitude,
                          decreasing = T),]

latsort[latsort$country == "USA","locality"]
```

The problematic records (1-3 above) are from Universities and botanical gardens. Records 4 and 5 above seem to be within the Caribbean distribution of *E. nocturnum*. Note that record 6 (Hawaii) doesn't have any information in the locality field that would identify it as an outlier. Visual inspection is critical in identifying outliers like these.

Based on the locality info within the points above, we may be able to search for similar records that don't look like outliers on the map by searching for words like "garden", "greenhouse", or "university" using ``grepl``. Keep in mind that we should search for these terms in other languages, too (in this case, Spanish and Portuguese).

```{r}
bad_words = c("greenhouse", "invernadero", "estufa",
              "garden", "jardin", "jardim", "university",
              "universidad", "universidade")

bad_words = c(bad_words, firstup(bad_words)) 

occ_total[grepl(paste(bad_words,
                      collapse = "|"),
                occ_total$locality), "locality"]
```

OK, only two more from Mexico, and both of those are from a natural reserve, so they're probably fine. Let's add the points in Hawaii, Colorado, Missouri, and Connecticut to a list of records to remove using the key as a unique identifier.

```{r}
bad_locals = latsort[latsort$country == "USA","key"][c(1:3, 6)]
```

Now, just in case, let's check to make sure none of these orchids were sampled from a movie theater.

```{r}
bad_words = c("movie theater", "cine", "cinema")

bad_words = c(bad_words, firstup(bad_words))

occ_total[grepl(paste(bad_words,
                      collapse = "|"),
                occ_total$locality), c("locality")]
```

There's always one. Add it to the list and remove!

```{r}
bad_locals = c(bad_locals, occ_total$key[grepl("cinema",occ_total$locality)])

occ_total = occ_total[!(occ_total$key %in% bad_locals),]
```

All of this underscores the importance of manually looking at locality data when possible. Many points will pass all of our previous cleaning steps, look fine on a map, and still come from unnatural or dubious environments. For relatively small datasets like this one, it's actually feasible to look at all of the locality information (with the help of Google Translate). For larger datasets, this level of scrutiny would not be worthwhile.

Finally, lets compare the number of occurrences from our initial query to the number in our final dataset.

```{r}
final_species_table = table(occ_total$name)
init_species_table; final_species_table
```
