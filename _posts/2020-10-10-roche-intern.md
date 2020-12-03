---
layout: post
title: Cancer drug response prediction 
date: 2020-10-10 22:39
category: Industry, Machine Learning, Bioinformatics
author: 
tags: [Internship]
summary: Modeling cancer drug response through informative gene expression and clinical data
---

Modeling cancer drug response through informative gene expression and clinical data
 **Note**: The project was compeleted as a statitical programming analyst intern in Roche(China)

*Manager: Zhanjian Dong*

*Mentor: Liming Li*  

*Intern: Shuangyu Xiong* 

## Introduction

### Background

1. Gene Expression has been shown to be able to infer how cancer cells respond to drugs
   > "Not only specific features can be discovered as biomarkers for resistance or sensitivity to a particular drug, but combinations of those same genomic and molecular features can be used to predict the effect of a drug on a patient)"  
   >
   > -- Azuaje, F. Computational models for predicting drug responses in cancer research. *Brief. Bioinform.* **18**, 820–829 (2017).

2. A large dataset
   *EDIS CIT Datamart* contains gene expressions and clinical data for research and drug development.

3. Machine learning approaches
   May be suitable to handle large data with high dimension.

4. Motivation:
   We can utilize *CIT Datamart* to predict patient’s possible response to drugs.

<!--more-->
### Example

- Given a patient with Lung cancer, we have the record of his/her demographic characteristics, such as *age, height, weight* and so on.
- Besides, we have tracked some clinical data from a series of medical examinations, such as *Systolic Blood Pressure* and whether it is normal, too high or too low.
- Then, based on above data and the patient’s gene expression, e.g. Engseq, we hope to find the pattern of the progression of the disease
- To sum up, we use the past data including clinical data and gene expression to predict whether the patient’s disease will be stable, progressive or complete in the future. 

## Workflow

1. Scientific Question

   Can we engineer new features from clinical data and high-dimensional genomic data (gene expression) which can better predict clinical response?

2. Overview

   Take response, clinical data and gene expression (RNAseq) for a few thousand patients, and build neural networks.

3. Data: 7 CIT studies

   - Clinical datasets: Analysis datasets (*Clinical Response, Baseline, Labs, Vitals, Tumor, Cancer Type*, *etc*)

   - Genomic datasets: Gene expression(RNASeq): N ~=*4,000* patients

4. Goal

   - Build neural network and compare its performance to logistic regression

   - Stretch: beat logistic regression by a couple % points

5. Methods

   ![methods](/../../media/2020-10-10-roche-intern/image-20201203230232448.png)

## Methodology

### Preprocess

1. Gene expression:  

   - Cut out gene with standard deviation equals to 0

   - Agglomeration & Standardized

2.  Response: Encode

3. Clinical data:

   - Data cleaning: drop missing values, drop features with only one class, combine several minority classes
   - Relevance analysis: distribution visualization, correlation analysis

### Feature Enineering

1. Categorical variables: calculate ratio, discretization 

2. Numerical variables: filtering

3. Combine several features to create a new feature to reduce dimension

4. Final input for model: 

   ![image-20201203230434416](/../../media/2020-10-10-roche-intern/image-20201203230434416.png)

   Dataset split: 

   ![image-20201203230456270](/../../media/2020-10-10-roche-intern/image-20201203230456270.png)

### Classification

1. LogisticRegression(CV) 

   - Sigmoid function g(z)

     ■$\theta$ is parameter vector

     ■$X$ is input vector

     <img src="/../../media/2020-10-10-roche-intern/image-20201203231731062.png" alt="image-20201203231731062" style="zoom:40%;" />

2. Tree-based model

   - Decision tree

     ![image-20201203232329461](/../../media/2020-10-10-roche-intern/image-20201203232329461.png)

     - Algorithm: 
       - Greedy: best split(gini, entropy, error)

     - Stop: 
       - all records belong to the same class
       - early stop (avoid data fragmentation, records in every leaf are too small)
       - when all records have similar attribute values

   - Ensemble

     - ![image-20201203232414458](/../../media/2020-10-10-roche-intern/image-20201203232414458.png)

     - bagging(random forest): Uniform sampling
     - boosting: Weighted sampling

3. Neural Network

   - MLP(multi-layer perception)
     - single input
     - multiple input

   - Batch normalization
   - Dropout

4. Parameter selection
   - Grid search
   - According to the learning curve

## Result

Accuracy(mean/std): 0.65

1. Conclusion
   - Neural Networks optimize the classification performance
   - Sorted feature importance uplifts interpretability
   - Ensemble model is more stable
2. Challenge
   - Imbalance data
     - Metrics: f1 score or recall instead of accuracy
     - Tree model: weigh errors based on class imbalance
     - Dataset: upsampling/downsampling
   - High-dimension
     - Feature selector: 
       - use ML approaches select important feature(e.g. decision tree)
     - Feature engineering: 
       - Combine features: height/weight-->ratio
       - Discard irrelevant feature: use Pearson correlation
       - Agglomeration