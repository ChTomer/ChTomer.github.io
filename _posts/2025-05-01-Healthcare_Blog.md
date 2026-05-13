---
title: "Fraud Detection in Healthcare Industry"
date: 2025-05-01
layout: single
categories: [projects]
tags: [python, data-analysis, visualization, Machine Learning, Data Science]
author_profile: true
read_time: true
---

# Catching Fraud in the Healthcare System

<img src="/assets/images/Healthcare fruad illustration.png" width="600">

*Catching Healthcare Fraud: A Data-Science Walk-Through*


**Posted by Tomer Choresh** <br> 
*Published on: May 1, 2025*


---

## The High Cost of Fraud

The U.S. Department of Justice (DOJ) created the Health Care Fraud Unit in March 2007 to fight illegal billing [1]. Today the DOJ estimates that fraud drains about $100 billion every year, which amounts to roughly 10% of all U.S. healthcare spending [2].

During a two-week operation in late 2024 the DOJ charged 193 people, including doctors, nurses, and other licensed professionals, for fraudulent schemes worth roughly $2.75 billion in intended losses [3]. But some of these fraudulent schemes are never detected, and the people who perpetrate them are never charged with their crimes.

As a data enthusiast, I wanted to work on a real-world problem with a clear impact for my project. Here, my goal was to analyze historical claim data and predict whether a provider is potentially fraudulent. A system that alerts payers to fraudulent claims is of great value, as it has the potential to reduce the costs of fraud by billions of dollars.

I set out to see how far a simple Logistic Regression model could go in spotting fraudulent providers in this dataset. I handled the full pipeline for this project: data cleaning, EDA, feature engineering, modeling, evaluation and visualization. You are invited to read through my process and explore the insights I gained along the way.

---

## Data and Tools

The project uses [Kaggle's Medicare fraud dataset](https://www.kaggle.com/datasets/rohitrox/healthcare-provider-fraud-detection-analysis/data). I focused on four of the eight files, joined on the `BeneID` and `Provider` primary keys.

| File | Columns | Rows | Purpose |
|---|---|---|---|
| Train-1542865627584.csv | 2 | 5,410 | Yes/No fraud labels by provider |
| Train_Outpatientdata-1542865627584.csv | 27 | 517,737 | Outpatient claim data |
| Train_Inpatientdata-1542865627584.csv | 30 | 40,474 | Inpatient claim data |
| Train_Beneficiarydata-1542865627584.csv | 25 | 138,556 | Beneficiary demographics (masked) |

For this project, I used Python and its tools and libraries. Specifically, I used the following:

- Jupyter Notebook
- Pandas
- NumPy
- Matplotlib & Seaborn
- Scikit-learn (+ imbalance-learn for SMOTE)

All code and commits are available in my [GitHub repository](https://github.com/ChTomer/MyCapstone).

---

## Exploratory Data Analysis

Before diving into the dataset, it's important to highlight a key point about my analysis. It was all based on aggregating at the **provider level**. The reason for that is the structure of the _“Train-1542865627584.csv”_ file, which has only two columns:

1. **Provider** - 5,410 unique IDs.
2. **PotentialFraud** - ‘Yes’ and ‘No’ labels.

The label breakdown is _90.65% non-fraud vs 9.35% fraud_, matching the DOJ estimate.

<img src="/assets/images/prop fraud.png" width="600">

*Fraudulent providers 9.35%, non-fraud 90.65%.*

Although fraudulent providers make up only 9.35% of the list, they account *over half of the total reimbursements*, as shown in the table below.

| PotentialFraud | IPInscClaimAmtReimbursed | OPInscClaimAmtReimbursed | Total | Share % |
|---|---|---|---|---|
| No | 167,008,510 | 93,853,510 | 260,862,020 | 46.9% |
| Yes | 241,288,510 | 54,392,610 | 295,681,120 | 53.1% |
| Total | 408,297,020 | 148,246,120 | 556,543,140 | 100% |

> *Takeaway: a small fraction of providers is responsible for the majority of billed dollars - an early hint that provider-level features will matter.*

---

## Feature Engineering

Because the label is assigned to the provider, every transformation keeps a single row per provider. I verified the row count after each join. Fraud at the provider level typically appears as a pattern across multiple claims rather than as isolated incidents.

Two signals stood out during EDA:

1. The total treatment time a provider bills.
2. The number of claims they submit.

These were calculated separately for inpatient (IP) and outpatient (OP) data, which resulted in four new features:

| Feature | Description |
|---|---|
| IPClaimDurationSum | Sum of inpatient claim duration |
| IPClaimDurationCount | Number of inpatient claims |
| OPClaimDurationSum | Sum of outpatient claim duration |
| OPClaimDurationCount | Number of outpatient claims |

The chart below compares the *average claim duration* for fraud vs non-fraud providers, split by IP and OP. In both IP and OP sets, fraudulent providers bill for noticeably longer sessions.

<img src="/assets/images/avg dur tre double.png" width="600">

*Average claim duration by fraud status (IP and OP).*

---

# Modeling

## Logistic Regression - Take 1

I began with baseline Logistic Regression (`scikit-learn` default). The accuracy and ROC-AUC look strong, but a recall of 0.38 means the model misses 65 of 105 fraudulent providers. That’s an unacceptable cost in a fraud-detection setting, where false negatives matter most. Because false negatives are so costly, I needed a better approach.

**Here are the metric results for LR (`scikit-learn` default):**

- Accuracy: 0.93
- Precision: 0.75
- Recall: 0.38
- F1 Score: 0.51
- AUC LR Model: [0.9239]
  
<img src="/assets/images/LR 1.png" width="600">

*Confusion matrix and ROC curve for baseline Logistic Regression.*

---

## LR + Hyperparameter Tuning Using GridSearchCV - Take 2

I ran a `GridSearchCV` over regularization strength (`C`), penalty type, and solver. The best combinations I found were:

{'C': 0.01, 'penalty': 'l1', 'solver': 'liblinear'}

| Metric | Take 1 (old) | Take 2 (new) |
|---|---|---|
| Accuracy | 0.93 | 0.92 (–0.01) |
| Precision | 0.75 | 0.81 (+0.06) |
| Recall | 0.38 | 0.28 (–0.10) |
| F1 | 0.51 | 0.41 |
| ROC-AUC | 0.9239 | 0.9257 |

I got better results for the Precision, but recall dropped by 0.10 points. As a result, 76 of 105 fraudulent providers still slip past from the true value. At this point, I noticed something odd: no matter what I tried, accuracy hovered around 90%. Then it struck me that the data were about 90% non-fraud. That’s why the model could predict 'clean' every time and still achieve a high accuracy score.

<img src="/assets/images/GSCV2.png" width="600">

*Confusion Matrix and ROC curve for Logistic Regression after hyper-parameter tuning with GridSearchCV.*

---

## LR + SMOTE - Take 3

This realization led me to the most important concept I applied to this imbalanced dataset. Once I understood that a model could predict 'non-fraud' every time and still score 90% accuracy, I knew I had to find a better solution. I researched methods for handling imbalanced datasets and discovered Synthetic Minority Over-sampling Technique (SMOTE) [4].

The SMOTE method generates synthetic data for the minority class until we get the same number of values for both the fraud and non-fraud labels as shown below.

> *Disclaimer: SMOTE does not create duplicate samples; it generates synthetic “neighbors” based on the existing minority class examples. I have attached below an example GIF that illustrates how it works.*

| Fraud label count | Before SMOTE | After SMOTE |
|---|---|---|
| Non-fraud (0) | 3,923 | 3,923 |
| Fraud (1) | 405 | 3,923 |

<img src="/assets/images/smote_demo.gif" width="600">

--- 
*Illustration of SMOTE on a graph.*

Running SMOTE with Logistic Regression gives me the following results:

- <mark>Accuracy: 0.90</mark>
- <mark>Recall: 0.86</mark>
- <mark>Precision: 0.48</mark>
- <mark>F1 Score: 0.61</mark>
- <mark>AUC with the SMOTE Model: [0.9611]</mark>

<img src="/assets/images/SMOTE3.png" width="600">

*Confusion Matrix and ROC curve for Logistic Regression with SMOTE (best results).*

This time I was much happier with my results! Finally, the recall increased to 0.86, and the ROC score increased to 0.9611. Precision fell to 0.48. Even though we had some more false alarms, it was worth it to catch most fraud.

As mentioned at the beginning of this blog, the cost of missing fraud is so significant that insurance companies would rather investigate more legitimate cases than risk missing fraudulent providers.

In summary, this table gives a short view of the three model performances:

| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | Note |
|---|---|---|---|---|---|---|
| Baseline LR | 0.93 | 0.75 | 0.38 | 0.51 | 0.924 | Imbalance hurts recall |
| GridSearch LR | 0.92 | 0.81 | 0.28 | 0.41 | 0.926 | No real lift |
| <font color="red">**SMOTE + LR**</font> | <font color="red">0.90</font> | <font color="red">0.48</font> | <font color="red">0.86</font> | <font color="red">0.61</font> | <font color="red">0.961</font> | <font color="red">Best ROC-AUC</font> |

---

## Key Findings

- Claim duration: a strong fraud signal that may be a sign of suspicious activity.
- Custom features: improved model performance.
- SMOTE + LR: great combination for imbalanced data.

---

## Summary

For this kind of dataset, I can explore more models, such as Random Forest, LightGBM, decision tree, etc.

Another concept I would love to explore is real-time fraud detection scenarios. Lastly, I would like to explore unsupervised learning methods. Autoencoders, in particular, could help reduce noise and highlight patterns within the minority fraud class.

This project reinforced how important it is to look beyond basic metrics like accuracy, especially when working with real-world, imbalanced datasets. Through feature engineering, model tuning, and SMOTE, I learned not just how to build better models, but how to think critically about the problem itself. I'm excited to keep exploring advanced techniques and contribute to meaningful solutions in the field of fraud detection and beyond.

---

## Resources

- [1] [Combating Health Care Fraud: 2024 National Enforcement Action](https://www.justice.gov/archives/opa/blog/combating-health-care-fraud-2024-national-enforcement-action#:~:text=Thursday%E2%80%99s%20announcement%20underscores,most%20egregious%20offenders)
- [2] [CRM 500-999. “Health Care Fraud - Generally”](https://arc.net/l/quote/aemwqblk)
- [3] [National Health Care Fraud Enforcement Action Results in 193 Defendants Charged and Over $2.75 Billion in False Claims](https://arc.net/l/quote/xcskoaji)
- [4] [Best Techniques And Metrics For Imbalanced Dataset](https://www.kaggle.com/code/marcinrutecki/best-techniques-and-metrics-for-imbalanced-dataset)
- [SMOTE By Code](https://machinelearningmastery.com/smote-oversampling-for-imbalanced-classification/)
- [SMOTE In General](https://www.youtube.com/watch?v=K-Q6ahRqxck)
- [Kaggle Link For The Dataset](https://www.kaggle.com/datasets/rohitrox/healthcare-provider-fraud-detection-analysis/data)
- [GitHub Repository For My Analysis](https://github.com/ChTomer/MyCapstone)
- [My Presentation’s Slides](https://drive.google.com/file/d/1jPnMJNZRz9oQVxCP3VmEwOjEiFoHIbL3/view?usp=drive_link)
- [My LinkedIn Profile. Please feel free to connect](https://www.linkedin.com/in/tomer-choresh/)  🙂
- Cover picture generated with Open-AI (GPT:4o + DALL-E)