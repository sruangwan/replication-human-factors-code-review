# Replication Package for "The Impact of Human Factors on the Participation Decision of Reviewers in Modern Code Review"
Shade Ruangwan, Patanamon Thongtanunam, Akinori Ihara, and Kenichi Matsumoto

## 1) Bibtex

```bibtex
@article{Ruangwan2017,
    author = {Ruangwan, Shade and Thongtanunam, Patanamon and Ihara, Akinori and Matsumoto, Kenichi},
    title = {The Impact of Human Factors on the Participation Decision of Reviewers in Modern Code Review}
    journal = {Under Major Revision at Empirical Software Engineering}
}
```

## 2) Download Processed Datasets

Each dataset contains 13 studied metrics. Each row also includes review ID and person ID (reviewer ID), and a participation decision of the reviewer (Review_Decision column).

- Android dataset (csv file, ~9 MB)
- OpenStack dataset (csv file, ~32 MB)
- Qt dataset (csv file, ~15 MB)

You can download these datasets [here](https://github.com/sruangwan/replication-human-factors-code-review/releases/latest).

## 3) Additional Results

## 4) Example R Scripts

### 4.1) Install Packages

```R
#For the model construction and analysis process
install.packages("rms")

#For the model analysis process
install.packages("doParallel")
install.packages("pROC")
install.packages("caret")
```

### 4.2) Model Contruction

```R
#Load RMS package
library(rms)
```
The output is:
```
## Loading required package: survival
## Loading required package: Formula
## Loading required package: lattice
## Loading required package: Hmisc
## Loading required package: ggplot2
## 
## Attaching package: ‘Hmisc’
## 
## The following objects are masked from ‘package:base’:
## 
##     format.pval, round.POSIXt, trunc.POSIXt, units
## 
## Loading required package: SparseM
## 
## Attaching package: ‘SparseM’
## 
## The following object is masked from ‘package:base’:
## 
##     backsolve
```
```R
#Load RMS package
df <- read.csv("{PATH_TO_DATASETS}/openstack.csv")

#Set dependent variable
df$y = df$"Review_Decision" == 1

#Select independent variables
ind_vars <- c("Familiarity_between_the_Invited_Reviewer_and_the_Patch_Author", "Median_Number_of_Comments", "Patch_Size", "Reviewer_Code_Authoring_Experience", "Reviewer_Reviewing_Experience", "Number_of_Remaining_Reviews", "Number_of_Concurrent_Reviews", "Review_Participation_Rate", "Number_of_Received_Review_Invitations", "Patch_Author_Code_Authoring_Experience", "Patch_Author_Reviewing_Experience", "Is_Core")

#Set data distribution for RMS package to construct a model
dd <- datadist(df[,c("y",ind_vars)])
options(datadist = "dd")

```
#### (MC1-a) Remove highly-correlated independent variables
```R
#Calculate spearman's correlation between independent variables
vc <- varclus(~ ., data=df[,ind_vars], trans="abs")

#Plot hierarchical clusters and the spearman's correlation threshold of 0.7
plot(vc)
threshold <- 0.7
abline(h=1-threshold, col="red", lty=2)
```
![](figures/examples/varclus-1.png)
```R
#Remove the highly correlated variable from the selected independent variables
reject_vars <- c('Number_of_Remaining_Reviews')
ind_vars <- ind_vars[!(ind_vars %in% reject_vars)]

#Re-calculate spearman's correlation between independent variables
vc <- varclus(~ ., data=df[,ind_vars], trans="abs")

#Re-plot hierarchical clusters and the spearman's correlation threshold of 0.7
plot(vc)
threshold <- 0.7
abline(h=1-threshold, col="red", lty=2)
```
![](figures/examples/varclus-2.png)

#### (MC1-b) Remove redundant independent variables
```R
red <- redun(~., data=df[,ind_vars], nk=0) 
print(red)
```
The output is:
```
## Redundancy Analysis
## 
## redun(formula = ~., data = df[, ind_vars], nk = 0)
## 
## n: 466520 	p: 11 	nk: 0 
## 
## Number of NAs:	 0 
## 
## Transformation of target variables forced to be linear
## 
## R-squared cutoff: 0.9 	Type: ordinary 
## 
## R^2 with which each variable can be predicted from all other variables:
## 
## Familiarity_between_the_Invited_Reviewer_and_the_Patch_Author 
##                                                         0.114 
##                                     Median_Number_of_Comments 
##                                                         0.045 
##                                                    Patch_Size 
##                                                         0.001 
##                            Reviewer_Code_Authoring_Experience 
##                                                         0.114 
##                                 Reviewer_Reviewing_Experience 
##                                                         0.138 
##                                  Number_of_Concurrent_Reviews 
##                                                         0.162 
##                                     Review_Participation_Rate 
##                                                         0.035 
##                         Number_of_Received_Review_Invitations 
##                                                         0.253 
##                        Patch_Author_Code_Authoring_Experience 
##                                                         0.141 
##                             Patch_Author_Reviewing_Experience 
##                                                         0.134 
##                                                       Is_Core 
##                                                         0.114 
## 
## No redundant variables
```
```R
#If there are any redundant variables, remove the redundant variables from the selected independent variables
reject_vars <- red$Out
ind_vars <- ind_vars[!(ind_vars %in% reject_vars)]
```
#### (MC2-a) Estimates a budget for degrees of freedom
```R
#Print number of FALSE and TRUE instances
print(table(df$y))
```
The output is:
```
# FALSE   TRUE 
# 44593 421927
```
```R
#Calculate and print the budgeted degrees of freedom
budgeted_df =  budgeted_df = floor(min(nrow(df[df$y == T,]), nrow(df[df$y == F,]) )/15)
print(budgeted_df)
```
The output is:
```
# [1] 2972
```
#### (MC2-b) Measure and plot a dotplot of the Spearman multiple ρ<sup>2</sup> of each independent variable
```R
sp <- spearman2(formula(paste("Review_Decision" ," ~ ",paste0(ind_vars, collapse=" + "))), data= df, p=2)
plot(sp)
```
![](figures/examples/spearman-dotplot.png)
#### (MC2-c) Allocate degrees of freedom based on the Spearman multiple ρ<sup>2</sup> of independent variables and fit a nonlinear logistic regression model using restricted cubic splines with original dataset
```R
model <- lrm(y ~ Familiarity_between_the_Invited_Reviewer_and_the_Patch_Author + Median_Number_of_Comments + Patch_Size + Reviewer_Code_Authoring_Experience + Reviewer_Reviewing_Experience + Number_of_Concurrent_Reviews + rcs(Review_Participation_Rate, 3) + Number_of_Received_Review_Invitations + Patch_Author_Code_Authoring_Experience + Patch_Author_Reviewing_Experience + Is_Core, data=df, x=T, y=T)
```
### 4.3) Model Analysis

#### (MA1) Evaluate the performance of the nonlinear logistic regression model
```R
#Load doParallel package
library(doParallel)
```
The output is:
```
## Loading required package: foreach
## foreach: simple, scalable parallel programming from Revolution Analytics
## Use Revolution R for scalability, fault tolerance and more.
## http://www.revolutionanalytics.com
## Loading required package: iterators
## Loading required package: parallel
```
```R
#Select the number of cores to use for parallel execution
#The example uses 4 cores
cl <- makeCluster(4)
registerDoParallel(cl)

#Define a function to use for combining outputs of the parallel execution
comb <- function(...) {
    mapply(rbind, ..., SIMPLIFY=FALSE)
}

#Run the parallel process with 1,000 iterations of foreach with the ```comb``` combination function
bootstrap_output <- foreach(i=1:1000, .combine='comb', .multicombine=TRUE) %dopar% {
    #Set seed so the results are reproducible
    set.seed(i)

    #Load library for each parallel process
    library(rms)
    library(pROC)
    library(caret)

    #Randomly draw a bootstrap sample with replacement from the original dataset and put it in ```training```
    #This will be a training dataset for the nonlinear logistic regression model
    indices <- sample(nrow(df), replace=TRUE)
    training <- df[indices,]

    #Put other instances that are not in ```training``` to ```testing```
    #This will be a testing dataset for the nonlinear logistic regression
    testing <- df[-unique(indices),]

    #Fit a nonlinear logistic regression model using the same allocated degrees of freedom as MC3-c using the training dataset
    m <- lrm(y ~ Familiarity_between_the_Invited_Reviewer_and_the_Patch_Author + Median_Number_of_Comments + Patch_Size + Reviewer_Code_Authoring_Experience + Reviewer_Reviewing_Experience + Number_of_Concurrent_Reviews + rcs(Review_Participation_Rate, 3) + Number_of_Received_Review_Invitations + Patch_Author_Code_Authoring_Experience + Patch_Author_Reviewing_Experience + Is_Core, data=training, x=T, y=T)
    
    #Calculate Precision, Recall, and F-measure 
    prob <- rms:::predict.lrm(m, testing, type='fitted')
    confMatrix <- caret:::confusionMatrix(table(ifelse(prob > 0.5, 1, 0), testing$Review_Decision))
    precision <- confMatrix$byClass[['Precision']]
    recall <- confMatrix$byClass[['Recall']]
    Fmeasure <- 2 * ((precision * recall) / (precision + recall))

    #Calculate area under the ROC curve (AUC)
    auc <- auc(testing[,dep],prob)

    #Calculate Brier score
    brier <- mean((prob-testing$Review_Decision)^2)

    #Combine the performance estimates as a result of single iteration
    cbind(recall, precision, Fmeasure, auc, brier)
}
```
You can download the performance estimates [here](https://github.com/sruangwan/replication-human-factors-code-review/releases/latest).

#### (MA2) Estimate the power of explanatory
The first part of the example script is the same as MA1 until the line where we fit a model. We seperate the script for the purpose of clarification. However, MA1 and MA2 can be combined and executed at the same time.
```R
#Load doParallel package
library(doParallel)
```
The output is:
```
## Loading required package: foreach
## foreach: simple, scalable parallel programming from Revolution Analytics
## Use Revolution R for scalability, fault tolerance and more.
## http://www.revolutionanalytics.com
## Loading required package: iterators
## Loading required package: parallel
```
```R
#Select the number of cores to use for parallel execution
#The example uses 4 cores
cl <- makeCluster(4)
registerDoParallel(cl)

#Define a function to use for combining outputs of the parallel execution
comb <- function(...) {
    mapply(rbind, ..., SIMPLIFY=FALSE)
}

#Run the parallel process with 1,000 iterations of foreach with the ```comb``` combination function
bootstrap_output <- foreach(i=1:1000, .combine='comb', .multicombine=TRUE) %dopar% {
    #Set seed so the results are reproducible
    set.seed(i)

    #Load library for each parallel process
    library(rms)
    library(pROC)
    library(caret)

    #Randomly draw a bootstrap sample with replacement from the original dataset and put it in ```training```
    #This will be a training dataset for the nonlinear logistic regression model
    indices <- sample(nrow(df), replace=TRUE)
    training <- df[indices,]

    #Put other instances that are not in ```training``` to ```testing```
    #This will be a testing dataset for the nonlinear logistic regression
    testing <- df[-unique(indices),]

    #Fit a nonlinear logistic regression model using the same allocated degrees of freedom as MC3-c using the training dataset
    m <- lrm(y ~ Familiarity_between_the_Invited_Reviewer_and_the_Patch_Author + Median_Number_of_Comments + Patch_Size + Reviewer_Code_Authoring_Experience + Reviewer_Reviewing_Experience + Number_of_Concurrent_Reviews + rcs(Review_Participation_Rate, 3) + Number_of_Received_Review_Invitations + Patch_Author_Code_Authoring_Experience + Patch_Author_Reviewing_Experience + Is_Core, data=training, x=T, y=T)
    
    #Estimate the power of explanatory (Wald χ2) of each variable with its statistical significance (p-value)
    explantory_power <- anova(m, test='Chisq')
    ep_df <- as.data.frame(explantory_power)
    chisq <- t(ep_df["Chi-Square"])
    pvalue <- t(ep_df["P"])

    #Combine the power of explanatory and the statistical significance as a result of single iteration
    cbind(chisq, pvalue)
}
```
You can download the power of explanatory and the statistical significance [here](https://github.com/sruangwan/replication-human-factors-code-review/releases/latest).