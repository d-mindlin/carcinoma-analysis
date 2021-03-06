---
title: "Survival Analysis: Large-Bowel Carcinoma"
author: "Daniel Mindlin"
date: "December 1st, 2018"
---

#Introduction

The study in question for this project is*Surgical adjuvant therapy of large-bowel carcinoma: an evaluation of levamisole and the combination of levamisole and fluorouracil*, conducted in in 1989 by the Mayo Clinic.

A total of 401 eligible patients with resected stages colorectal carcinoma, a form of cancer, were randomly assigned to no-further therapy, to adjuvant treatment with either levamisole alone, or levamisole plus fluorouracil (5-FU). 

The data from the study is found in the *colon* dataset in the R package *survival*, while its constituent variables are:

*id*:	Each patient identification tag

*rx*:	Treatment Type - Observation (or control), Levamisole, Levamisole + 5-FU

*sex*:	Gender of patient 

*age*:	Patient age in years

*obstruct*:	Obstruction of colon by tumour

*perfor*:	Perforation of colon

*adhere*:	Adherence to nearby organs

*nodes*:	Number of lymph nodes with detectable cancer

*time*:	Days until event or censoring

*status*:	Censoring status

*differ*:	Differentiation of tumour (1=well, 2=moderate, 3=poor)

*extent*:	Extent of local spread (1=submucosa, 2=muscle, 3=serosa, 4=contiguous structures)

*surg*:	Time from surgery to registration (0=short, 1=long)

*node4*:	More than 4 positive lymph nodes

*etype*:	Event type: 1=recurrence,2=death

The available data begs for further questions. Whether or not a particular treatment type leads to lower probability of the cancer recurring. First, a model ought to test that alone for each treatment type, then expand the model to include other potential covariates from the dataset. 

Furthermore, it is of interest to examine the same effect from recurrence to death from cancer.

#Stage 1: Basic Model
##Estimation

First, it is prudent to compare three different Survival rates using Kaplan-Meier Estimators: the rate of all patients, and the rates for each treatment type.


The Kaplan-Meier estimator for the recurring chance of cancer, regardless of treatment, is shown by:

```{r}
colon <- na.omit(colon)
colon.est <- Surv(colon$time, colon$etype == 1)
colon.fit <- survfit(colon.est ~ 1)
plot(colon.fit, main = "Chance of Cancer Recurrance over Time", xlab
= "Time in Days of Treatment", ylab = "Proportion of Patients with Recurring Cancer")
```

Comparing the recurrance rate based on type of treatment is shown by:
```{r}
colon.treat <- survfit(colon.est ~ colon$rx)
plot(colon.treat, col=c(2,4,3), main = "Chance of Cancer Recurrance over Time", xlab
= "Time in Days of Treatment", ylab = "Proportion of Patients with Recurring Cancer")
legend(10,0.4,c("Observed","Levamisole","Levamisole + 5VU"),fill=c(2,4,3))
```
25% Quantile: 1183

25% Quantile: 2352

75% Quantile: 2802

##Regression Analysis:

To perform a likelihood ratio test whether treatment type has an effect on recurrance rate, we use a Cox Proportional Hazard function to fit *time* and *etype* to *rx*.

```{r}
colon.cox.fit <- coxph(colon.est ~ colon$rx)
```

The p-value for the likelihood ratio test is 0.006, much smaller than 0.05, indicating that treatment type has an effect on recurrance rate.

The hazard proportion between the control group and the patients receiving solely Levamisole is 0.9313. This indicates that the probability of cancer recurrance is 6.88% lower among Levamisole patients than the control group. 

Similarly, the hazard proportion between the control group and the patients receiving Levamisole and 5-FU is 0.7792. This indicates that the probability of cancer recurrance is 22.08% lower among Levamisole and 5-FU patients than the control group. 

95% Confidence Interval for Parameter Estimate:
```{r}
betainterval <- coef(colon.cox.fit) + c(-1.96,1.96) * sqrt(colon.cox.fit$var)
```
For the first group, we yield an interval from -0.2373504 to -0.189273.

For the second group, we yield an interval from -0.1325358 to -0.08367566.

#Stage 2: Regression Model
##Regression Analysis:


Adding complexity to the model is prudent to control for variability and confounding. Adding the various cancer symptoms/complications and patient characteristics that *colon* contains to the model. Using the step() function in the R *LEAPS* library to use stepwise regression yields a model with the lowest AIC.

```{r, include=FALSE}
colon.cox.full <- coxph(colon.est ~ colon$rx + colon$age + colon$sex + colon$obstruct + colon$perfor + colon$adhere + colon$nodes + colon$differ + colon$extent + colon$surg)
colon.cox.AIC <- stepAIC(colon.cox.full, direction = "both")
```

This yields a reduced with lowest AIC, containing *rx*, *age*, *obstruct*, *nodes*, *extent*, and *nodes*:
```{r}
colon.cox.red <- coxph(colon.est ~ colon$rx + colon$age + colon$obstruct + colon$nodes)
```
The p-value for the reduced model's likelihood ratio test is 2e-16, much smaller than than the model with *rx* along (which had a p-value of 0.009), indicating that *rx*, *age*, *obtstuct*, and *nodes* has an effect on recurrance rate and much smaller than the model with only *rx*. 

The 95% Confidence Interval between Levamisole and the control is between 0.7762 and 1.0710.

The 95% Confidence Interval for Levamisole + 5FU and the control is between 0.6576 and 0.9101.

##Checking Assumptions:

The *cox.zph()* function uses Schoenflield Residuals with a global Chi-Square test to check the Proportional Hazards Assumption. 

```{r}
cox.zph(colon.cox.red, transform="km", global=TRUE)
```
Since both the global and per-variable P-values are all reasonably large, Cox PH is appropriate.

Log-log plot for the model:

```{r}
plot(colon.treat,
fun="cloglog",col=c(2,4,6),xlab="Survival Time",ylab="Probability",lwd=2, main = "Log-log plot")
legend("topleft",legend=c("Observation","Levamisole","Levamisole + 5FU"), 
pch = rep(15,4),col=c(2,4,6))
```

#Stage 3: Recurrant Events

An interesting subject to examine is the time between recurrance of cancer and patient death. To study this, a new dataset containing only deaths (*etype* = 2) while the time values changed to the time between observance of recurrance and time of death.
```{r}
t2 <- colon %>% filter(etype==2) %>% select(time)
t1 <- colon %>% filter(etype==1) %>% select(time)

colon.diff <- colon %>% filter(etype==2)
colon.diff$time <- (t2-t1)
colon.diff$time <- colon.diff$time$time
```

##Estimation

The new Kaplan-Meier estimator for chance of death after recurrance of cancer, regardless of treatment, is shown by:

```{r}
colon.diff <- na.omit(colon.diff)
colon.diff.est <- Surv(colon.diff$time, colon.diff$etype == 2)
colon.diff.fit <- survfit(colon.diff.est ~ 1)
plot(colon.diff.fit, main = "Chance of Death Since Recurrance", xlab
= "Time in Days of Treatment Since Recurrance", ylab = "Proportion of Patients Experiencing Death")
```

Comparing the death rate based on type of treatment is shown by:

```{r}
colon.diff.treat <- survfit(colon.diff.est ~ colon.diff$rx)
plot(colon.diff.treat, col=c(2,4,3), main = "Chance of Death Since Recurrance", xlab
= "Time in Days of Treatment Since Recurrance", ylab = "Proportion of Patients Experiencing Death")
legend(10,0.4,c("Observed","Levamisole","Levamisole + 5VU"),fill=c(2,4,3))
```
25% Quantile: 0

25% Quantile: 0

75% Quantile: 353

The preponderance of 0's in the data suggests that many patients were recording as deceased concurrently with diagnosis of recurrance. It is more imformative to simply remove these 0's and consider those patients who survived at least a day past recurrance.

```{r}
colon.death <- colon.diff %>% filter(time != 0)
```

Our new Kaplan-Meier curves for both overall time from recurrance to death and based on treatment type:

```{r}
colon..diff <- na.omit(colon.death)
colon.death.est <- Surv(colon.death$time, colon.death$etype == 2)
colon.death.fit <- survfit(colon.death.est ~ 1)
plot(colon.death.fit, main = "Chance of Death Since Recurrance", xlab
= "Time in Days of Treatment Since Recurrance", ylab = "Proportion of Patients Experiencing Death")

colon.death.treat <- survfit(colon.death.est ~ colon.death$rx)
plot(colon.death.treat, col=c(2,4,3), main = "Chance of Death Since Recurrance by Treatment", xlab
= "Time in Days of Treatment Since Recurrance", ylab = "Proportion of Patients Experiencing Death")
legend(10,0.4,c("Observed","Levamisole","Levamisole + 5VU"),fill=c(2,4,3))
```

25% Quantile: 182.0

25% Quantile: 356.5

75% Quantile: 695.5

##Regression Analysis:

Building a model to predict death rate based on the information at hand can once again be performed by AIC. 
```{r, include=FALSE}
colon.death.full <- coxph(colon.death.est ~ colon.death$rx + colon.death$age + colon.death$sex + colon.death$obstruct + colon.death$perfor + colon.death$adhere + colon.death$nodes + colon.death$differ + colon.death$extent + colon.death$surg)
colon.death.AIC <- stepAIC(colon.death.full, direction = "both")
colon.death.red <- coxph(colon.death.est ~ colon.death$rx + colon.death$age + colon.death$sex + colon.death$obstruct + colon.death$perfor + colon.death$nodes)
```
Our reduced model's predictors include: *rx*, *age*, *sex*, *obstruct*, *perfor*, and *nodes*.

The 95% Confidence Interval between Levamisole and the control is between 0.9634 and 1.495.

The 95% Confidence Interval for Levamisole + 5FU and the control is between 1.1210 and 1.838.

##Checking Assumptions:

The *cox.zph()* function uses Schoenflield Residuals with a global Chi-Square test to check the Proportional Hazards Assumption. We repeat this for our newest model. 

```{r}
cox.zph(colon.death.red, transform="km", global=TRUE)
```
Since both the global and per-variable P-values are all reasonably large, Cox PH is appropriate.

Log-log plot for the model:

```{r}
plot(colon.death.treat,
fun="cloglog",col=3:5,xlab="Survival Time",ylab="Probability",lwd=2, main = "Log-log plot")
legend("topleft",legend=c("Observation","Levamisole","Levamisole + 5FU"), 
pch = rep(15,4),col=3:5)
```



#Data Source
JA Laurie, CG Moertel, TR Fleming, HS Wieand, JE Leigh, J Rubin, GW McCormack, JB Gerstner, JE Krook and J Malliard. Surgical adjuvant therapy of large-bowel carcinoma: An evaluation of levamisole and the combination of levamisole and fluorouracil: The North Central Cancer Treatment Group and the Mayo Clinic. J Clinical Oncology, 7:1447-1456, 1989.

