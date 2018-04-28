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

We will be using ClimNA as our example for this workshop, which ***only works for occurrences in North America***. Be aware that there are other resources available for **South America** and **Europe**: [https://sites.ualberta.ca/~ahamann/data.html](https://sites.ualberta.ca/~ahamann/data.html).<br>These should have very similar functionalities to what we're learning in the workshop.

**WorldClim** - [http://worldclim.org/](http://worldclim.org/)

***

## ClimNA example

You will need to download the data file called **dasypus_novemcinctus_gbif.csv**. This is a dataset of hummingbird clearwing moth (*Hemaris thysbe*) occurrences that I downloaded directly from GBIF's website and converted into a .csv file. We will clean it to prepare it for ClimNA using R. First, set your working directory if you have not already and load the required libraries. You will also need to install the R package dismo.

```{r}
setwd("~/Desktop/UFBI workshop example/");
library(scrubr);
library(readr);
#install.packages("dismo");
library(dismo);
```

Load the dataset you downloaded into R.

```{r}
library(readr)
hemaris_gbif <- read_csv("~/Desktop/UFBI workshop example/Armadillo/dasypus-novemcinctus-gbif.csv")
View(hemaris_gbif)
```
### Cleaning data for ClimNA

This dataset contains a lot more data than we need for running ClimNA. Let's first transform this into a data frame and extract only the necessary columns necessary: occurrenceID, publishing organization code, latitude, longitude, elevation, and event date.

```{r}
data <- data.frame(hemaris_gbif)
hemaris <- data[,c(3,16,17,18,21,25)]
View(hemaris)
```

![Image of Screen after this step](https://lh3.googleusercontent.com/mnUnNg5fV_C6jMnoCmUArq1TlP1DBFIx1eU_kiX3eTuykylqSkwNSgbLtAiOR8P29Nc-vQTDt8-nTr56wZrAq7EW0u9V9ypNgS6x5r3-OpYXutYbJnrdCzWoziT9F7kuFP8bnL80bQ44Ooy_JHQ7wmWJyZiJtYPCyBbRJcMCGbjjlwZyYxxEGZ5jSxO3Em2dOJH6wQukjmxK6mx47nh9g52PGZMzw0vjKfGjmukM_vKXaa8svfPj7S_QFI10zw1W61bx1XVAJ4gW7kFW3Znl_Y9XAC1QQ0EzQ51Hb7SYh44oeMWKKSKE9CLGiJhVwq1cMHUQWRxPw1uSlgKse9WuKZOzGq3vY7XVRN1cocEn7uz7GAwrkF3Ns9b-w0JgefNrMYsrhC8t6D9rrI0prhwmZPM2WC-6iHGYID62foMG-2O9th14b1pTzAHUQ--6TGY7_6dJo6q1mbPAeQc9OMrQdb-1b_xdymI79ACPrcn73MaQGN_jBruVWPgJFtWhSaNbktazRYLibqnMPlN08QkpGnmMmWzEN5dtTLn0IO-Gmcetgh6CqflRXUkMztXx4QnWMrFcn5ZJ_HZ6ComTwAHxGRV2X5pOIocR0rfNaOyEsr3U3LD7yZ9WrwtlwHEo1U21LCityKLqGkyAVfWOZ8PCjWDtCSUc-Ng7=w1141-h471-no)

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

At this step, it is important to open the **Hemaris-clean-dates.csv** file to check when our occurrences come from. For our particular example, the occurrences range from 1969 to 2017. Unfortunately, ClimNA only has data up until 2013. For simplicity's sake, it is probably best to delete the two occurrences that fall outside of ClimNA's range (shown highlighted in the image below).

<img src="https://lh3.googleusercontent.com/ffl5mW_x-L399HY3xu70OPznF_WGLqbnoCUflALsn0Ndxvt42_J_Fw5dnaku0Mg4BqG1N4mrJAqxQC9qPpjFhjGJqSoP3qcDebbh72L5ye3ed40AiX9r8TiZhzMtL3B_l1MFNp6IxVMeVEwJzeywCZYUhVFhgaM12ika89zGrLZLqKJ1pOW4btexYrFFIY_Llmqu9uBaVsoFgofgjPpdTXpjtZhp_L4PCk-BGlgP26gq1pcyPNk4Hj92hATpE3XYVk4JvUqK9Il6h9hjULrma3A4JyAwkzu1J2HYIYqLJYXzaxVv5Qg9Kp3RNLhGYFCV-VoUr1RcMqwP7vkfd_AsfIbVgsXkdMU-ukr8JYz5tkjBX3x9c1KujY0kkk-k8awKxZMHFnqIzMv3J2VQ_8f-3NEL0Dr0AtkbLF3sTuNANEZMRQyn7hWvR7AsFTKQhUvonhnuFM18TcevB7Yyfj9hKahbPjE68JcNQ4QuOPvYFI2O48SXJD8G-Xxw1Pfz0POU3A0yqMWsPD3gQyOL7ll6Ay-fZAVobcwdyV4tKmvrhUyXOXgTumjRub-joR3YLIuWWatJTgILINlYzHZcUIv0oFJn08rUYnM9Dpp1oRM=w934-h592-no" width="550" height="330"/>

Note the ID1 and ID2 for these records, and delete them from the **Hemaris-ClimNA.csv** file and save it. You will need to save a blank spreadsheet as a .csv file for your output from ClimNA. Fill out the ClimNA interface with the appropriate input, output, and year range. We will discuss more about these options in the workshop. Click "Calculate TS".

<img src="https://lh3.googleusercontent.com/49r0_aaufHLwBoCIbDYxIicobpLoRV1kl48k8rEt3hAUe0YkVYFuvDgiL9o3Y1Bz2ydPZ7t72xEvir56SyGjwQU_ppz-5x_PqHo5qFKiVk7zoXbBspPf4osF4JYxnCspfx9oBmst5RcOYuVruHvsuFKVvqKY_cgewQf6JSKA6MUP55ufSxCRiR-of2J5-LgqeMCEkcGZdinOxAF_HwBYmnzTdFwKxxn0jKbOWMb_HR3D1QL__CAPqe1aEbj_i0lKxNOofQMXPLTBQFDykAGSpGGv8NSvunwp_2ZFOtydTf9Yf6jD7dS7ASdV27GB3f7OD8v1e8RuHPrBDc9Fwq96P5N93p_oXRhc8Ict4wjPj0yOB9zxQVe_AvOTnjljKjGNa5i_FhbZ2qXCDfvKbOVo-Yjg5bkD5bAY837QEiGJv3wWOO6eDEumcv25Gs9EkEDzHrOjFuoSoy2ybuI2hCDoh3VSoalSBMu24uSgyI574H2aVPI6hNoxPbGUWnrggypD_A3pNPN-6FfjnQkylCAxdfMdp1oDbsh2ZalAWi-q2Ib0D-_SLL_8nEBi2YQJ1gUMD8FEc4aOXjJTG6SZJVS-J0v_097pg_OVcuayPrA=w1130-h1134-no" width="430" height="450"/>

If you open your output file, it should now have data in it. Because we ran these as a time series, you'll notice that all of our occurrences now have a row of data with each variable from ClimNA for every month of every year from 1969-2013. The definitions of these annual climate variables can be found in the **help.rtf** ClimNA file.

<img src="https://lh3.googleusercontent.com/jd7eNxuHlll_w9iCvH3AgcJupWeXPxx5uZMdei5ObPIKug3pV0XgBXyBsvrGyfnq_gSzGqzHOh-rOGv4bu64iwLxRV1BepvoAwnLpBGfFs_EHgBoaW2-443uU45sadreRakMk6WSNf31rVGeZ60_dWcGl_Kty0-bF2AUNmpG9ZrCou8qlqpmvpyd4WR4lLj1fwyq9CNWK_Ez4IPA7hLHjiUErf0EFMSp8881p6ruvbxaIG7K_8zo1rD7WoYznv8MW0AItq-8wHPCVh2rO6yP5zAbNB9zJnkiE2j_1ggGXwjLgJrLiaEWQe02sNQAPkMtk4v2bg0DGOJ9e3CNS5g9kkm8RPAafGuRY9nyVpEx_9OYfXWEmsXWhhkoUsvN5ujzvKo9nHkZiV8MDV3Kba9KJNTXqDF_oTTb14VL5pH41yjiHnlKEYAYIs-EWP4i0en0xgUWtb9a5i9HTrZBgSlU3NnoR62mJegmxSyqZVl9kvUIjvCoS3KZnNnJs39xwogL8WiNjnZv9uE_pZC8vbafr7v5x6njkZ1wmZfNEy6jiQTC4sDua9I_pHpt5Q9QI-LN_JLm-CByONtDT8ZlPn3oGQetnrfUWasZIXdINLg=w842-h672-no" width="510" height="480"/>

### Cleaning the ClimNA output

Really, we only need the associated climate data from the year that the occurrence was recorded. In order to keep this example simple, we will only pull the MAT (mean annual temperature) and MAP (mean annual precipitation) for the year we need, and then we will use the biovars function to create climate variables from the data we have.
