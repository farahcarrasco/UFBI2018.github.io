---
title: "Workshop Outline"
layout: page
permalink: /Outline/
---

## UFBI 2018 Spatial Ecology Workshop

The two-day workshop will take place on **May 8th and 9th**, and will be divided into four sessions of introductory, hands-on presentations. The workshop is free, but **subscription** is required. Lunch and breaks provided by the University of Florida Biodiversity Institute.


#### May 8th: Occurrence and Climate data from big dataset

| Time  | Activity |
| ------------- | ------------- |
| 9:00 am - 10:30 am  | Acquiring occurrence data from online databases - *Jon Spoelhof*  |
| 10:30 am - 10:45 pm  | BREAK  |
| 10:45 pm -12:15 pm | Occurrence data continued - *Jon Spoelhof*  |
| 12:15 pm - 1:15 pm  | LUNCH  |
| 2:45 pm - 3:00 pm  | Climate data resources overview - *Laura Brenskelle*  |
| 3:00 pm - 3:15 pm  | BREAK  |
| 3:15 am - 4:45 pm  | Climate data hands-on examples - *Laura Brenskelle*  |
| 4:45 pm - 5:00 pm  | Summary for the day  |

<br><br>


#### May 9th: Spatial Ecology from Remote sensing

| Time  | Activity |
| ------------- | ------------- |
| 9:00 am - 10:30 am  | Raster images and estimation of forest cover in R - *Farah Carrasco-Rueda*  |
| 10:30 am - 10:45 pm  | BREAK  |
| 10:45 pm -12:15 pm | Use LiDAR point-cloud data to delineate crowns - *Farah Carrasco-Rueda*  |
| 12:15 pm - 1:15 pm  | LUNCH  |
| 2:45 pm - 3:00 pm  | 	Species classification using NEON hyperspectral data. PART 1 - *Sergio Marconi* |
| 3:00 pm - 3:15 pm  | BREAK  |
| 3:15 am - 4:45 pm  | Species classification using NEON hyperspectral data PART 2 - *Sergio Marconi*  |
| 4:45 pm - 5:00 pm  | FINAL REMARKS |

<br><br>


## In Detail

**Jon Spoelhof**
1.	Obtaining occurrence data in R
2.	Taxonomic name resolution using the taxise package
3.	Occurrence data queries using the spocc package
a.	Sources of data: record types, limitations, and overlap
b.	Source-specific query options
c.	Strategies for integrating raw data from multiple sources
4.	Data cleaning (dplvr, raster, rgdal, and maps packages)
a.	Removing dubious records and missing data
b.	Removing duplicate records
c.	Spatial filtering based on geographic boundaries
d.	Strategies to exclude unnatural occurrences
<br><br>

**Laura Brenskelle**
1.	Review different climate data resources
a.	ClimNA - https://sites.ualberta.ca/~ahamann/data/climatena.html
b.	Future data projections - https://cds.nccs.nasa.gov/nex-gddp/
c.	WorldClim - http://worldclim.org
o	NOAA - https://www.giss.nasa.gov/tools/panoply/
d.	Their different temporal and spatial resolutions and why these matter
2.	WorldClim data example
3.	How to extract estimates from ClimNA
4.	NOAA data brief example (six-hour temporal resolution)
o	Download Panoply - https://www.giss.nasa.gov/tools/panoply/download/
<br><br>

**Farrah Carrasco-Rueda**
1.	Raster image visualization in R
a.	How to enter raster images in R
b.	Change the projection for the images
2.	Reclassification of original information. Packages: raster, rgdal, and SDMTools
3.	Estimate forest cover
a.	Crop polygons from rasters. Function maptools
b.	Estimate the amount of forest cover inside the polygons
4.	Moving window
5.	How to use LiDAR point-cloud in R
a.	Quick intro to LiDAR (5 mins.)
b.	LiDAR visualization
c.	Classification of points
d.	Points normalization
e.	Data visualization (1 or 3 channels)
<br><br>

**Sergio Marconi**
1.	Crown Delineation:
a.	Create a Canopy Height Model out of the LiDAR point-cloud data
i.	Using a smooth filter: why we want to “reduce the sharpness”
ii.	Apply Silva et al., 2016 to estimate Individual Tree Crowns (ITCs)
iii.	The magic of other languages under the hood: polygonise with Python in R
b.	Extract data from hyperspectral raster for a bunch of crowns
2.	Species classification
a.	Data manipulation
b.	Application of a Random Forest Classifier
c.	What’s next? An insight on deep learning for species classification 
<br><br>

