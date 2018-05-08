---
layout: page
title:  Climate Data
permalink: /Laura/
---

# Download the necessary tools
***Please download these BEFORE attending the workshop on May 8. If you have problems, you may contact me via lbrensk@ufl.edu with questions.***

**ClimNA** - You can download the program and necessary files here: [https://sites.ualberta.ca/~ahamann/data/climatena.html](https://sites.ualberta.ca/~ahamann/data/climatena.html)

If you have a Windows machine, you should be able to simple unzip the folder and run the "ClimateNA_v5.21.exe" file.
If you have a Mac, you may require a few other programs to run ClimNA, which is made for Windows. You will need to install [Wine](http://www.winehq.org), which allows you to run Windows apps without installing Windows. I recommend downloading Wine itself from the [winehq website](http://www.winehq.org), but the directions included [here](https://www.davidbaumgold.com/tutorials/wine-mac/#part-1:-install-homebrew) about installing Homebrew and XQuartz, which may be necessary in order to run Wine, are very helpful. Finally, you may need to run the ClimateNA file called “Libraryfiles.exe” in Wine before you are able to run the "ClimateNA_v5.21.exe" file itself.

**Panoply** - You can download this program here: [https://www.giss.nasa.gov/tools/panoply/](https://www.giss.nasa.gov/tools/panoply/)

***

## Climate data resources overview

We will be using ClimNA as our example for this workshop, which ***only works for occurrences in North America***. Be aware that there are other resources available for **South America** and **Europe**: [https://sites.ualberta.ca/~ahamann/data.html](https://sites.ualberta.ca/~ahamann/data.html)<br>These should have very similar functionalities to what we're learning in the workshop.

**WorldClim** - [http://worldclim.org/](http://worldclim.org/)

**ECMWF** - Hourly data; [http://apps.ecmwf.int/datasets/data/interim-full-daily/levtype=sfc/](http://apps.ecmwf.int/datasets/data/interim-full-daily/levtype=sfc/) May need to create an account to download datasets.
<br>Other ECMWF datasets: [http://apps.ecmwf.int/datasets/](http://apps.ecmwf.int/datasets/)
***

## ClimNA example

You will need to download the data file called **hemaris_thysbe-gbif.xls** from the Google Drive: [https://drive.google.com/open?id=1CbBDCiBKxBoWokZc40LJo5nmwDZf7hBl](https://drive.google.com/open?id=1CbBDCiBKxBoWokZc40LJo5nmwDZf7hBl). This is a dataset of hummingbird clearwing moth (*Hemaris thysbe*) occurrences that I downloaded directly from GBIF's website and converted into a .xls file. We will clean it to prepare it for ClimNA using R. First, set your working directory if you have not already and load the required libraries. You will also need to install the R package dismo.

```{r}
setwd("~/Desktop/UFBI workshop example/");
library(scrubr);
library(readr);
#install.packages("dismo");
library(dismo);
```

Load the dataset you downloaded into R.

```{r}
library(readxl)
hemaris_gbif <- read_excel("~/Desktop/UFBI workshop example/hemaris_thysbe-gbif.xls")
View(hemaris_gbif)
```
### Cleaning data for ClimNA

This dataset contains a lot more data than we need for running ClimNA. Let's first transform this into a data frame and extract only the necessary columns necessary: occurrenceID, publishing organization code, latitude, longitude, elevation, and event date.

```{r}
data <- data.frame(hemaris_gbif)
hemaris <- data[,c(3,16,17,18,21,25)]
View(hemaris)
```

![Image of Screen after this step](https://ufbi2018.github.io/figures/ClimNAcleaning.png)

Now we are going to clean up our 1421 records to narrow it down further. We only want occurrence records that have data in all of the fields we've specified in our search.

```{r}
cleaned_occ <- (as.data.frame(hemaris)[complete.cases(as.data.frame(hemaris)), ])
```

Next, let's clean these data to get rid of incomplete coordinates, unlikely coordinates, and duplicate records.

```{r}
scrubbed <- coord_incomplete(cleaned_occ)
scrubbed_data <- coord_unlikely(scrubbed)
unique_data <- unique(scrubbed_data)
View(unique_data)
```
You'll notice even after these simple cleaning steps, we have gone from 1421 records to only 16, but for our purposes here, this is ideal.

Next, we will have to do some reformatting because ClimNA is particular about the formatting of the .csv you give it.

```{r}
names(unique_data) <- c("ID1", "ID2", "lat", "long", "el", "date")
View(unique_data)
```
ClimNA is very picky about formatting. Above, we renamed our columns to suit the required formatting:<br>
occurrenceID -> ID1<br>
publishingorgkey -> ID2<br>
decimallatitude -> lat<br>
decimallongitude -> long<br>
elevation -> el<br>

The two ID fields are important if you want to go back to the larger spreadsheet that we cleaned to view or verify the rest of the associated information with the specimens you are using in your analysis.

ClimNA itself does not care about or want the eventDate -> date information. However, we will need it for processing post-ClimNA so we will save a copy of these data with the date column intact, then delete this column and save the dataset to be put into ClimNA.

```{r}
write.csv(unique_data, "Hemaris-clean-dates.csv", row.names = F)
unique_data$date <- NULL
View(unique_data)
write.csv(unique_data, "Hemaris-ClimNA.csv", row.names = F)
```
### Running ClimNA

Now we are actually ready to run the ClimNA program with the data we've cleaned. Open the **ClimateNA_v5.21.exe** file on your computer. If you have a Mac and have not installed Wine or the other necessary programs to run this Windows program, see at the top of this page because you will experience difficulties at this step. **This is why it is important to do this BEFORE coming to the workshop!** Otherwise, we will waste time troubleshooting Wine installation.

At this step, it is important to open the **Hemaris-clean-dates.csv** file to check when our occurrences come from. For our particular example, the occurrences range from 1969 to 2017. Unfortunately, ClimNA only has data up until 2013. For simplicity's sake, it is best to delete the two occurrences that fall outside of ClimNA's range (shown highlighted in the image below).

<img src="https://ufbi2018.github.io/figures/RemoveRange.png" width="550" height="330"/>

Note the ID1 and ID2 for these records, and delete them from the **Hemaris-ClimNA.csv** file and save it. You will need to save a blank spreadsheet as a .csv file for your output from ClimNA. Fill out the ClimNA interface with the appropriate input, output, and year range. We will discuss more about these options in the workshop. Click "Calculate TS".

<img src="https://ufbi2018.github.io/figures/ClimNAscreenshot.png" width="430" height="450">

If you open your output file, it should now have data in it. Because we ran these as a time series, you'll notice that all of our occurrences now have a row of data with each variable from ClimNA for every month of every year from 1969-2013. The definitions of these climate variables can be found in the **help.rtf** ClimNA file.

<img src="https://ufbi2018.github.io/figures/Monthlyvariables.png" width="510" height="480">

### Cleaning the ClimNA output

Really, we only need the associated climate data from the year that the occurrence was recorded. In order to keep this example simple, we will only pull the average temperature for each month for the year we need, and then we will use the biovars function to create climate variables from the data we have.

```{r}
all_data <- read.csv("hemaris-output-monthly.csv")
clean_dates <- read.csv("Hemaris-clean-dates.csv")
```
Somehow ClimNA managed to garble some of observational data's identification codes. We'll fix that because we need them to be the same to match between the output and the spreadsheet with the dates.

```{r}
levels(all_data$ID1)[levels(all_data$ID1)=="2.7E+67"] <- "27e66ae6-7b71-4f15-bd0e-052a9cf67afa"
levels(all_data$ID1)[levels(all_data$ID1)=="8"] <- "08BBLEP-01675"
all_data$ID1 <- as.character(all_data$ID1)
all_data$ID1[all_data$ID1 == "10" & all_data$Elevation == "1146"] <- "10BBCLP-0319"
all_data$ID1[all_data$ID1 == "10" & all_data$Elevation == "1137"] <- "10BBCLP-0320"
```

Next, we'll normalize the dates to make a separate year column.

```{r}
clean_dates$year <- format(as.Date(clean_dates$date, format = "%Y-%m-%d"), "%Y")
```
Next we'll use the IDs to match our data and create a spreadsheet with only our records and the average temperatures from each month in the occurrence year.

```{r}
subsetted_data <- all_data[all_data$ID2 == clean_dates$ID2 & all_data$ID1 == clean_dates$ID1 & all_data$Year == clean_dates$year & all_data$Latitude == clean_dates$lat & all_data$Elevation == clean_dates$el,c(1:30,43:54)]
merged_data <- merge(subsetted_data, clean_dates, by.x = c("Year", "Latitude", "Elevation", "ID1", "ID2"), by.y = c("year", "lat", "el", "ID1", "ID2"))
final_data <- merged_data[,c(4:6,2:3,44,7:42)]
write.csv(final_data, "hemaris-final-data.csv", row.names = F)
```
<img src="https://ufbi2018.github.io/figures/ClimNA-output.png">

### Using bivoars to create bioclimatic variables 

Now, let's use the biovars function from the dismo package to create bioclimatic variables for each of our occurrences.

```{r}
x = 1
iterations = 14
variables = 19
biovarsoutput <- matrix(ncol=variables, nrow=iterations)
repeat {
occ <- as.numeric(final_data[x,7:42])
tmax <- occ[1:12]
tmin <- occ[13:24]
precip <- occ[25:36]
biovarsoutput[x,] <- biovars(precip, tmin, tmax)
x = x+1
if (x == 14) {
break
}}
biovarsoutput <- as.data.frame(biovarsoutput)
names(biovarsoutput) <- c("bio1", "bio2", "bio3", "bio4", "bio5", "bio6", "bio7", "bio8", "bio9", "bio10", "bio11", "bio12", "bio13", "bio14", "bio15", "bio16", "bio17", "bio18", "bio19")
View(biovarsoutput)
write.csv(biovarsoutput, "hemaris_biovarsoutput.csv", row.names = F)
```

**Congratulations!** You have created your own customized bioclimatic variables for each occurrence that can be used in modeling.
