# Qualitative Activity Recognition on Weight Lifting Exercise
Danilo Carvalho  
July 16, 2014  

# Synopsis

## Background

Using devices such as Jawbone Up, Nike FuelBand, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement – a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how much of a particular activity they do, but they rarely quantify how well they do it. In this project, your goal will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here: [http://groupware.les.inf.puc-rio.br/har](http://groupware.les.inf.puc-rio.br/har) (see the section on the Weight Lifting Exercise Dataset).

## Data

The training data for this project are available here: 

[https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv)

The test data are available here: 

[https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv](https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv)

The data for this project come from this source: [http://groupware.les.inf.puc-rio.br/har](http://groupware.les.inf.puc-rio.br/har). If you use the document you create for this class for any purpose please cite them as they have been very generous in allowing their data to be used for this kind of assignment. 

## Goals  

1. The main goal of this project is to predict the manner in which they did the exercise.  
2. Describe how we built our model.  
3. How we used cross validation.  
4. What we think the expected out of sample error is.  
5. We will also use our prediction model to predict 20 different test cases.  

# Analysis

## Loading packages

First of all, we need to load the packages required to our analysis.


```r
library(knitr)
library(data.table)
library(caret)
```

```
## Loading required package: lattice
## Loading required package: ggplot2
```

## Reading in raw data

The following step consists in reading in the data from which we built our model.


```r
rawData <- fread("pml-training.csv", header = TRUE, na.strings = c("NA", ""))
```

## Remove index columns and NA columns

Before we start to try to build a model from this data, we have to be sure that its structure is formated into a convinient configuration that allow us to manipulate it..

Concretely, we will check its *structure* (frame, table, matrix and so on), *dimensions*, *variable names* and any *column which might have a negative impact in our analysis*.

Despite we do not intend to print the output of the whole data on this paper, we can see that each observation forms a row and each variable forms a column. Although we still have some meaningless columns in the data.  

* The first 7 columns are index columns.


```r
# Show index columns
head(rawData[, 1:7, with=FALSE], 3)
```

```
##    V1 user_name raw_timestamp_part_1 raw_timestamp_part_2   cvtd_timestamp
## 1:  1  carlitos           1323084231               788290 05/12/2011 11:23
## 2:  2  carlitos           1323084231               808298 05/12/2011 11:23
## 3:  3  carlitos           1323084231               820366 05/12/2011 11:23
##    new_window num_window
## 1:         no         11
## 2:         no         11
## 3:         no         11
```

```r
# Select index columns
indexCols <- names(rawData)[1:7]
indexCols
```

```
## [1] "V1"                   "user_name"            "raw_timestamp_part_1"
## [4] "raw_timestamp_part_2" "cvtd_timestamp"       "new_window"          
## [7] "num_window"
```

* There are 100 columns filled with *missing values*, as below.


```r
# Show NA columns
head(rawData[, 11:3, with=FALSE], 3)
```

```
##    total_accel_belt yaw_belt pitch_belt roll_belt num_window new_window
## 1:                3    -94.4       8.07      1.41         11         no
## 2:                3    -94.4       8.07      1.41         11         no
## 3:                3    -94.4       8.07      1.42         11         no
##      cvtd_timestamp raw_timestamp_part_2 raw_timestamp_part_1
## 1: 05/12/2011 11:23               788290           1323084231
## 2: 05/12/2011 11:23               808298           1323084231
## 3: 05/12/2011 11:23               820366           1323084231
```

```r
# Select the predictors without missing values
goodCols  <- colSums(is.na(rawData)) == 0
# Proportion of meaninful predictor
table(goodCols)
```

```
## goodCols
## FALSE  TRUE 
##   100    60
```

Thus we remove them (index and predictors with missing values columns) from the data.


```r
# Remove index columns and NA columns
rawData   <- rawData[, goodCols, with=FALSE]
rawData   <- rawData[, c(indexCols):= NULL]
```

## Splitting the data

The section comprises two steps:

### Split the data in a predictors (X) and response (Y) sets

Splitting the data in a predictor table and a outcome vector is not a fundamental step to build this model, since we could perform our analysis using the *formula* notation in our code. Although it proves to be a quite computing demanding practice. Thus we chose this structure to train our model.


```r
# Split the data in two sets: features (X) and response (Y)
Y <- factor(rawData[, classe])
X <- rawData[, -53, with=FALSE]
```

### Split the data in training and testing sets


```r
set.seed(2134)
inTrain <- createDataPartition(Y, p=0.75, list=FALSE)

trainX  <- X[c(inTrain),]
testX   <- X[-c(inTrain)]

trainY  <- Y[c(inTrain)]
testY   <- Y[-c(inTrain)]
```

In summary, we have:

* Trainining sample   
    * Predictors: `trainX`  
    * Response: `trainY`  

* Testing sample  
    * Predictors: `testX`  
    * Response: `testY`

## Preprocessing the data:

Before we try to fit a model to our data let's check its dimmentions.  

* Number of obsersations in the training sample: **14718**.  
* Number of observations in the testing sample: **4904**.  

* Number of predictors in both samples: **52**.  

As we can see, removing the *index* and *NA columns* give us 52 predictors instead of 159 predictors. Nevertheless, it remains a quite large number of variables. Therefore we can expect some correlation among the predictors. To estimate this correlation we use the `findCorrelation` function from `caret` package which takes at least two arguments: a correlation matrix and a absolute correlation value as threshold. To exemplify we use the `findCorrelation` to show the descriptors with absolute correlations above 0.75.


```r
XCor <- cor(X)
highCor <- findCorrelation(XCor, cutoff = 0.75)

# Number of predictors above the cutoff
length(highCor)
```

```
## [1] 20
```

```r
# Predictors above the cutoff
names(X)[highCor]
```

```
##  [1] "accel_belt_z"      "roll_belt"         "accel_belt_y"     
##  [4] "accel_arm_y"       "total_accel_belt"  "accel_dumbbell_z" 
##  [7] "accel_belt_x"      "pitch_belt"        "magnet_dumbbell_x"
## [10] "accel_dumbbell_y"  "magnet_dumbbell_y" "accel_arm_x"      
## [13] "accel_dumbbell_x"  "accel_arm_z"       "magnet_arm_y"     
## [16] "magnet_belt_y"     "accel_forearm_y"   "gyros_arm_y"      
## [19] "gyros_forearm_z"   "gyros_dumbbell_x"
```

### Dimension Reduction Methods and Principal Components Analysis

Principal Components Analysis (PCA) is a popular approach for deriving a low-dimensional set of features, also know as *principal components* with $k$-dimentions from a large set of variables, a $n$-dimentional set where $k < n$. PCA is therefore also referred to as a *dimensionality reduction* algorithm. Generally, when faced with a large set of correlated variables, PCA allow us to summarize this set with a smaller number of representative variables that collectively explain most of the variability in the original set.  

Formaly, PCA performs a set of eigenvector calculations, trying to identify the $k$-dimentions subspace in which the data approximately lies, returning the top k eigenvectors (denoted by `caret` package as PC1, PC2, ..., PC$k$) where as much as possible of this variance is still retained.  

To perform PCA using `caret` package, we use the preProcess function. Here we set the percent of variance captured by PCA at 90%.


```r
# Preprocess the data applying PCA
preProc <- preProcess(X, method="pca", thresh = 0.9)

PCAtrainX <- predict(preProc, trainX)
PCAtestX  <- predict(preProc, testX)
```


## Building the Model: Three Based Methods

### Overview

Three-based methods for classification involve stratifying or segmenting the predictor space into a number of simple regions. In order to make a prediction for a given observation, we typically use most commonly occurring class of training observations in the region to which it belongs. Since the set of splitting rules used to segment the predictor space can be summarized in a tree, these types of approaches are known as decision tree methods.

### Aggregating Three Methods: Random Forest

Tree-based methods are simple and useful for *interpretation*. However, they typically are not competitive with the best supervised learning approaches in terms of prediction *accuracy*. 

*Random forests* overcomes this problem producing multiple trees which are then combined to yield a single consensus prediction. Combining a large number of trees can often result in dramatic improvements in prediction accuracy, at the expense of some loss in interpretation.

### Pruning the threes  

To prune our Random Forest model we chose the *Cros-Validation* method instead of the default *Bootstrapping* method which concretely prodices the same level of *accurary* at a cost of a much longer computational time.  

Additionaly, for the $K$-fold CV, we set $k=4$. Although it may result in a higher bias estimetion, it provides a much more stable (lower variance) model.


```r
# Train the data using Random Forest
trControl <- trainControl(method = "cv", number = 4)
modFit <- train(PCAtrainX, trainY, method = "rf", trControl=trControl, importance=TRUE)
```

### Predict out-of-sample error


```r
# Predict out-of-sample error
testPred <- predict(modFit, newdata = PCAtestX)
matrixTest <- confusionMatrix(testPred, testY)

accuracyOut <- matrixTest$overall[1]

accuracyOut
```

```
## Accuracy 
##   0.9694
```

```r
outError <- 1 - accuracyOut
outError[[1]]
```

```
## [1] 0.03059
```

The estimated out-of-sample error is 1.000 minus the model's accuracy, the later of which is provided in the output of the confusionmatrix, or more directly via the 'postresample' function.

In order to estimate the the *out-of-sample error*, we take the complement of the *accuracy* of our model.

* Accuracy:  **97%**.
* Out-of-sample error:  **97%**.

***


```r
# Load newData data
newData <- fread("pml-testing.csv", header = TRUE, na.strings = c("NA", ""))

# Remove index columns and NA columns
indexCols <- names(newData)[1:7]
goodCols  <- colSums(is.na(newData)) == 0

newData   <- newData[, goodCols, with=FALSE]
newData   <- newData[, c(indexCols):= NULL]

# Remove problem index column
newData <- newData[, -53, with=FALSE]

# Preprocess the data applying PCA
PCAnewData <- predict(preProc, newData)

# Predict new sample error
newDataPred <- predict(modFit, newdata = PCAnewData)

# Save each prediction in a different text file 
pml_write_files = function(x){
  n <- length(x)
  for(i in 1:n){
    filename <- paste0("problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

pml_write_files(newDataPred)
```
