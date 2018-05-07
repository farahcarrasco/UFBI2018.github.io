---
layout: page
title:  Ecology with NEON AOP
permalink: /MarconiS/
---


## Crown segmentation and identification with LiDAR and Hiperspectral data

In this module we will learn to use LiDAR and hyperspectral data to automatically extract ecological information about individual trees at landscape scale. 


### Before starting, you will need:
* Intermediate knowledge of R
* **[Download](https://drive.google.com/drive/folders/1f8UK2zYaQxF8KE7MUIvsp4M63aDtAveW?usp=sharing)** the dataset we'll use for this lesson
* have the following libraries installed:
  1. `raster`
  2. `rgdal`
  3. `readr`
  4. `dplyr`
  5. `tidyr`
  6. `e1071`
  7. `devtools`
  8. `rlas`
  9. `lidR`
  10. `FactoMineR`
  11. `caret`

* We suggest you to have the libraries `rlas` and `lidR` installed from latest version on github. 


#### Delineation from LiDAR module:
- Visualize LiDAR data
- Create a Canopy Height Model (CHM) from point-cloud dataset
- Perform crown delineation using Silva et al., 2016
- Extract and save single crowns data

#### Species classification from hyperspectral module:
- Pre-processing of hyperspectral pixels
- Stratified sampling by species 
- Applying a Support Vector Machine (SVM) to the data (the easy way)
- (tuning SVM parameters with 10-fold Cross Validation (10k-CV))
- Crown predictions and confusion proportions

### Some background



### Visualize LiDAR data

First of all, we will need our dataset to play with. You can download it from **[here](https://drive.google.com/open?id=1UA-0TVt1V-r3hnOVi5IvIUFk7KJq9lVO)**


```{r}
devtools::install_github("Jean-Romain/rlas", dependencies=TRUE)
devtools::install_github("Jean-Romain/lidR", dependencies=TRUE)
library(rlas)
library(raster)
library(lidR)
library(readr)
library("rgdal")
```

Next step, we want to identify the path of the data to be loaded. In this first section we'll load a LiDAR point-cloud file `.laz`, an hyperspectral image `.tif`, and some ground data tree polygons `.shp`. First, let's point to the path where the data is sitting. 

Then, we'll get the name of the files we are interested to load. To have it automatically done, we'll use the function `list.files`, which returns a character vector, with the name of each single file respecting a defined `pattern`

```{r}
pt <-  "./inputs/delineation/"
f <- list.files(pt, pattern = ".laz")[1]
f_hps <- list.files(pt, pattern = "hps.tif")[1]
```
In this case we chose to store only the first file name found by `list.files`. That's why we used [1] after the function.

Let's now load a point-cloud dataset, and normalize it. Why are we normalizing? Point cloud data is potentially prone to a lot of noise. That noise can partly be reduced if we reduce squeeze the variance relative to the vertical gradient of the points bouncing back. Fortunately, the `lidR` comes with an handy function, `lasnormalize`, which does it for us:

```{r}
las = readLAS(paste(pt, f, sep=""))
lasnormalize(las, method = "knnidw", k = 10L)
```
Now, we can plot our pointcloud data for a cool 3D view!

```{r}
plot(las, color="Intensity", colorPalette = terrain.colors(50), trim=0.95)
```
![classified_las](figures/lidar_perspective.png)
 

Not happy with the color palette? you can actually take advantage of the hyperspectral information to add a cool false color palette to your point-cloud view. Let's see how.
Let's first load our data as a raster stack. Raster stack are way faster than common rasters and allow us to work with rasters with a very high number of bands.We can plot, for example, the combination of band 16, 86 and 177 using 'plotRGB'

```{r}
hps   = stack(paste(pt, f_hps, sep =""))
plotRGB(hps, r=16, g=177, b=86, stretch="lin")
image_rgb <- stack(hps[[86]], hps[[177]],hps[[16]])

```

![false_color_hps](figures/false_color_hsp.png) 

Not bad, right? they seem good enough to color our point cloud pixels. Let's extract and normalize them: 

```{r}
ch1 = image_rgb@layers[[1]]
ch2 = image_rgb@layers[[2]]
ch3 = image_rgb@layers[[3]]
ch1[ch1 >10000] = 0 
ch2[ch2 >10000] = 0 
ch3[ch3 >10000] = 0 
ch1 = stretch(ch1)
ch2 = stretch(ch2)
ch3 = stretch(ch3)
```

Now, let's reclassify three new channels, so that they have the information about the color intensity in our composite RGB

```{r}
lasclassify(las, ch1, "R")
lasclassify(las, ch2, "G")
lasclassify(las, ch3, "B")
las@data <- las@data[!is.na(las@data$R)]
```
And now, the magic! let's colorize our point-cloud with the three channels just realized!

```{r}
lascolor(las, nbits = 8)
plot(las, color = "color")
```
![classified_las](figures/lidar_false_color.png)


### Create a Canopy Height Model 

Now that we have point cloud data we can use that directly to predict crowns shape and position, or we can produce a raster file, to be used to perform crown delineation with other types of algorithms. NEON produces already canopy height models, but of "only" 1m2 resolution. 
Don't you want more? Let's check how to get a 0.5m2 resolution CHM!

```{r}
chm = grid_canopy(las, res = 0.5, subcircle = 0.2, na.fill = "knnidw", k = 4, p = 1)
chm = as.raster(chm)
plot(chm)
```
![chm_unsmoothed](figures/chm_no_blur.png)
 

Unfortunately, LiDAR point cloud data may be noisy. This can be a problem especially if we go so high resolution. In fact, is a good practice to filter blur the image, just a little, avoiding big artifacts and unnatural crowns

```{r}
kernel = matrix(1,3,3)
chm = raster::focal(chm, w = kernel, fun = mean)
chm = raster::focal(chm, w = kernel, fun = mean)
plot(chm)
```

![chm_smoothed](figures/chm_blur.png)

### Perform crown delineation using Silva et al., 2016

Now, let's try to apply a crown delineation algorithm (Silva et al., 2016). First of all we will need to look for the highest points, which will represent the top of our trees. Let's look at what are the parameters involved:

```{r}
ttops = tree_detection(chm, 5, 2)
```

5 and 2 seem a reasonable way to go, for this case. At this point we can predict our crowns' shape!

```{r}
crowns <-lastrees_silva(las, chm, ttops, max_cr_factor = 0.6, exclusion = 0.5, extra = T)
plot(crowns, col = random.colors(length(crowns)))
```
![crowns](figures/itcs_1km.png)

### Extract and save single crowns data

Now, we happily have produced a raster with single crowns, which you can transform in a polygon layer, if you want (for time sake, that's not something we'll do here).
*What can we do with these crowns, though?*

Most of the time, in Forestry and Ecology, we collect a sample of data, and that's what we use for assessments. *What if we could align data collected on the field with the remote sensing data, and use their combination to have better more comprehensive information?* 
Well, it would be awesome, no!? Let's see how to!
First of all, let's load part of the data published from a recent Data Science Evaluation (ref):

```{r}
itcs<-readOGR(pt, "ground_data")
itcs <- crop(itcs, extent(chm))
plot(itcs, add=T)
```
![itc_overimposed](figures/species_with_itc.png)


Those are crowns polygons collected on the field, with associated species identifier. Let's say, we want to extract reflectance and height data to be used to build a statistical model to classify all the crowns we have produced before!
To extract information from a raster, we'll use the function `extract`

```{r}
crown_data_chm <- extract(chm, itcs)
```
Noooow, is time to store the data extracted. you need to unlist them, and recursively append it to your final data. You could use a more elegant (potentially faster) style using functions of the `apply` family, instead of a for cycle. For simplicity though, and given the relatively little data involved, let's use a for cycle:

```{r}
canopyHeight <- data.frame(matrix(NA, nrow=0, ncol = 2))
colnames(canopyHeight) <- c("ITC, average_H")
for(itc_id in 1:length(crown_data_chm)){
  print(itc_id)
  temp <- c(itc_id, max(unlist(crown_data_chm[itc_id])))
  canopyHeight <- rbind(canopyHeight, temp)
}
head(canopyHeight)
write.csv(canopyHeight, "./data/canopyHeight.csv")
```

And here we go! your csv has a column of crown identifier taken from the ground, along with a column with the average height!

#### CHALLENGE: Calculate NDVI from hyperspectral and plot it!

Now that you know how to extract values from a raster, try to calculate the NDVI from the hiperspectral image and store it in a csv file. Remember, the bands used to calculate the NDVI in hierspsectral setting are `band_90` and `band_58`

## Species classification from hyperspectral module:

Now that you know how to extract data for each single crown, and know how to link information from the ground to the produced polygons, you have everything you need to build a statistical model to automatically determine which species each crown in the area is. In this submodule you'll learn to deal with hiperspectral data, how to correctly divide your dataset to avoid overfitting, and how to aggregate your results to individual crown level. First, you need to load some libraries to manipulate your data and build/test a Support Vector Machine (SVM)

```{r}
library(dplyr)
library(e1071)
library(tidyr)
library(FactoMineR)
library(caret)
```

Now, let's load some data extracted already in a csv form. These data have the crown identifier, the species, tree height, and spectral reflectance in 426 bands for a bunch of pixels (and 305 individual trees). Data are publicly available as part of a data science competition (Marconi et al., in prep). The whole dataset can be downloaded at: LINK, but for the sake of this workshop, just use the data in the inputs folder. 

First step, let's import our features (*hyper_bands_train.csv*) and response variables (*species_id_train.csv*)

```{r}
# load a complete dataset
pt <-  "./inputs/classification/"
features <- read_csv("./inputs/classification/hyper_bands_train.csv")
head(features)
speciesID <- read_csv("./inputs/classification/species_id_train.csv")
```

We can look now at the reflectance spectra of a pixel:

```{r}
species <- unique(speciesID$species)
aPixel_spectra <- features[1,3:428]
plot(as.numeric(aPixel_spectra), type = "l")
```
![reflectance_raw](figures/spectra_raw.png)


As you can see, these data are far from perfect. Especially, there are some regions that are absorbing all the light, no matter which pixel are we in. Those bands are known as **water absorption bands**, and we want to get rid of them. Fortunately, they are always the same, for they depend on hte chemistry of water! We can then point to the `bad bands` and pull them out.

```{r}
bandsMetadata <- read_csv("data/classification/hyper_bands.csv")
goodBands <- which(bandsMetadata$Noise_flag==0)+2
goodBands <- c(1:2, goodBands)
features <- features[,goodBands]
plot(as.numeric(features[1,3:371]), type = "l")
```
![cleaned_spectra](figures/spectra_clean.png)

We basically figured which bands are to be saved, including bands 1 and 2, which are crown identifier and canopy height model. The plotted bands, now, look definitively better!

Now, in the whole scene we may have pixels that are darker, pixels that are withina crown but mostly representing wood, soil, or other stuff than leaves. We want to get rid of those, because the relationship we are looking for is maily attributable to leaves reflectance spectra. In this, the same index we learned about this morning, the NDVI, may come handy. Let's poolish our data of all those pixels with NDVI less than 0.3, that is, all those data point which are far from being green leaves.

```{r}
ndvi <- (features$band_90- features$band_58)/(features$band_58 + features$band_90)
features[which(ndvi < 0.3),]=NA
features <- features[complete.cases(features), ]
```
Now, we have a lot of features, most of which, you can tell, are highly correlated. It is good in this case to reduce the number of dimensions by applying a data transformation which can held most of the variability in the data in a way lower number of dimensions. A commonly used technique for that is the PCA.

```{r}
pca_summary <- PCA(features[-1])
pca_features <- cbind(features[1], pca_summary$ind$coord)
```
![pca1](figures/pca_components.png) ![pca2](figures/pca_position.png)

Now, extremely important when using machine learning methods, is to divide your dataset into a portion used to build the model, and a fully independent dataset, to test if our model is working fine. Those are called respectively `trainset` and `testset`. Generally, you may want to divide the `trainset` in `train` and `validation`, whihc in this case is automatically happening when using cross validation. 

Further problem, our dataset is not well balanced: 70% of our trees are *Pinus palustris* (PIPA), and each species may have a different average number of pixels per crow because of the canopy structure. We want to sample our dataset in a **stratified** way. In short, we want to select crowns to be in a ratio of 0.8 to 0.2 stratified by species.

```{r}

# divide data in training, validation, and test
length(unique(features$crown_id))
set.seed(1)
test_data <- speciesID %>% 
  group_by(species_id) %>%
  sample_frac(0.2)
train_data <- speciesID[!(speciesID$crown_id %in% test_data$crown_id), ]
```

Now, that we know which crown have been divided in the train-test split, let's extract the features associated to those crowns. To do so, we'll use the function `inner_join`, from `dplyr`, which is doing something like the traditional `vlookup` in excel.

```{r}
#create augmented matrixes
test_set <- inner_join(test_data, pca_features, by = "crown_id")
train_set <- inner_join(train_data, pca_features, by = "crown_id")
```
And voila! here is the head of our trainset!

```{r}
head(train_set)[,1:6]
```

Now, we need to create a dataframe where the function svm knows who is the x, and who is the y, especially ina case where our features, the x, is a matrix of 370 columns! Another tricky thing to remember: svm works with numbers, not with characters! but we need to tell our svm we don't want to perform a regression, but classify between species! How to? let's treat our character labels as `factors`:

```{r}
train_set$species_id <- factor(train_set$species_id)
y <- (train_set$species_id)
x <- data.matrix(train_set[,-(1:2)])
```
Oooook, everything is ready! Let's build our simple SVM out of the train data!

```{r}
model_svm <- svm((y) ~ x)
```
Now, let's see what our model has memorized in his algorithm!

```{r}
pred_train <- predict(model_svm, data = x)
#Plot the predictions and the plot to see our model fit
plot(y, pred_train)
confusionMatrix(y, pred_train)

```
![untuned_train_svm](figures/train_svm.png)


We can more intuitively look at what is the accuracy of our model, by calculating the times it gets the right answers, out of thetotal number of pixels in the training set:

```{r}
svm_accuracy = sum(y == pred_train)/length(y)
svm_accuracy

# [1] 0.8798367
```

Usually, you may want to explore the parameterization space to get the potential best options, yes, but is good practive to do so on different combinations of data. Why is that? because the model may be fitting on the data, and not get realistic patterns. Usually, to reduce that problem, we divide the traininset in 10 folds, and sequenially build a svm with a bunch of different parameters combiantions on 9 folds, and validate on the remaining 1. This way, we can calculate among the different combinations, which one is performing the best on the training-valiadtion set, accounting for all 10 combinations.
In SVM, that can be done using the function `tune`:

```{r}
svm_tune <- tune(svm, train.x = x, train.y = y, gamma = 2^(-1:1), cost = 2^(2:4))
best_mod <- svm_tune$best.model
best_mod_pred <- predict(best_mod, data = x) 
plot(y, best_mod_pred, pch=4)
confusionMatrix(y, best_mod_pred)

```
![tuned_train_svm](figures/train_cv_svm.png)

Let's see how much it is performing on the training set:
```{r}
svm_accuracy = sum(y == best_mod_pred)/length(y)
svm_accuracy

# [1] 0.9204828
```

Not bad! But remember: that is what the model has **memorized**, not necessarily what it **learned**! What does it mean? Simply that the model used the same data to build a relationship between the species labels and the features provided. It memorized what's the pattern observed for these data, adn is prone to over perform on them. Sometimes, it may have learned patterns that don't even realy exist! That is why it is important to test our model on new data, and see if what it says it learned, is not the Christmas poem!

```{r}
x <- data.matrix(test_set[,-(1:2)])
y_test <- factor(test_set$species_id)
```
#now, let's see how the two perform on an independent test dataset

```{r}
pred_test <- predict(model_svm, newdata = x)
plot(y_test, pred_test, pch=4)
confusionMatrix(y_test, pred_test)
```
![untuned_test_svm](figures/test_svm.png)

```{r}
svm_accuracy = sum(y_test == pred_test)/length(y_test)
svm_accuracy

# [1] 0.7101086
```
See? performance naturally go lower on independent data! 

```{r}
pred_best_mod_test <- predict(best_mod, newdata = x)
length(pred_best_mod_test)
plot(y_test, pred_best_mod_test, pch=4)
confusionMatrix(y_test, pred_best_mod_test)
```
![tuned_test_svm](figures/test_cv_svm.png)

```{r}
svm_accuracy = sum(y_test == pred_best_mod_test)/length(y_test)
svm_accuracy

# [1] 0.7126149
```
And supposedly, even if seems worse on the training set, using cross validation ensures better predictions on the test set!
That said, it is not that bad, is it?
However, don't forget one thing: these are pixel based results! We may be very good in getting the most common widest crowns, but very bad in anything else! Let's see how our SVM is performing on each single crown!

### Crown predictions and confusion proportions

First step, we want to calculate the frequencies at which each species is predicted within a crown. One way to do that, using the results of the model we have built already, is to aggregate pixel based results to crown level, and calculate the frequency of getting any species in them. To do so, we'll manipulate data using `dplyr`.

```{r}
#aggregate results to objects
results <- as.data.frame(cbind(test_set$crown_id, pred_test))
colnames(results) <- c("crown_id", "species_id")
prob_vector_test <- results %>%
  group_by(crown_id) %>%
  count(species_id) %>%
  mutate(freq = n / sum(n))
```

We want to make it in such a way that each species label is the same as the levels of our species_id factor, and that it is in a `data.frame` where each row represents a secific crown, and each column the frequency of pixels belonging to a particular species. Again, we'll use 'dplyr' data manipulation skills to have it done in a few lines.

```{r}
prob_vector_test$species_id <- levels(pred_test)[prob_vector_test$species_id]
# create confusion matrix
frequency <- prob_vector_test %>%
  select(-(n))%>%
  spread(crown_id, freq) 
frequency <- t(frequency)
colnames(frequency) <- frequency[1,]
frequency <- frequency[-1,]
```
Remember, order in this case matters! you can end up comparing different trees if you don't check how the labels are ordered in your observed vs modeled sets.
```{r}
test_data$crown_id
prob_vector_test$crown_id
test_data<-test_data[order(test_data$crown_id),]
```

Finally let's see: how good is our model in getting individual tree crowns classification?
```{r}
majority_class <- apply(frequency,1, which.max)
majority_class <- colnames(frequency)[majority_class]
svm_accuracy = sum(test_data$species_id == majority_class)/length(test_data$species_id)
svm_accuracy

# [1] 0.8032787
```
Weeeell you see? crowns based predictions boost up our rank-1 accuracy a lot, right? Yet, there is a bunch of other cool stuff you can do to make it better, like reduce the amount of predictors by performing PCA, normalize the variance of our predictors, or even chose a more promising algorithm! Now you have the data: you can play to find a better solution than this baseline! 

Last piece of details: try to calculate the confusion matrix:
```{r}
confusionMatrix(test_data$species_id, majority_class)
```
it returns `Error: `data` and `reference` should be factors with the same levels.`
Why is that? well, for once, in your R experience, factors are good! it is not clear to the computer how to treat these characters, hence it needs it to be factorized. However, it is important to make sure the labels and their order is the right one. To do so, specify the labels to be the species in the **training** set:

```{r}
crowns_observed_factor <- factor(test_data$species_id, levels = sort(unique(test_data$species_id)))
crowns_predicted_factor <- factor(majority_class, levels = sort(unique(test_data$species_id)))
caret::confusionMatrix(crowns_observed_factor, crowns_predicted_factor)
plot(crowns_observed_factor, crowns_predicted_factor)
```
