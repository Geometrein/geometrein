---
author: "Tigran Khachatryan"
date: 2022-02-12
linktitle: Time Series Forecasting with SARIMAX
menu:
  main:
    parent: tutorials
title: Time Series Forecasting with SARIMAX
weight: 10
draft: false
tags: ["wolt", "python", "statmodels"]
categories: ["Data Science", "Python", "Time Series"]
---

## Exploratory Data Analysis of Food Delivery Sales Data

In this article we will implement a SARIMAX model in order to forecast the sales of a food delivery company.
This is a short version of the analysis the more in depth version and the code can be found in this repo:

The underlying sales dataset contain two months of sales data generated in Helsinki, Finland. The dataset has the following variables:

![dataframe_info](../../images/wolt_sarimax/dataframe_info.png)

Let’s look at the available data and see what aspects are covered by the dataset.
'
![df_corr_heatmap](../../images/wolt_sarimax/df_corr_heatmap.png)

**Observations**

- At a first glance, it seems that there are no immediately apparent relationships between the variables.
- ACTUAL_DELIVERY_MINUTES — ESTIMATED_DELIVERY_MINUTES is strongly correlated with ACTUAL_DELIVERY_MINUTES and ESTIMATED_DELIVERY_MINUTES but that doesn’t give us new or useful information.
- There are correlations between coordinates. Not a very useful relationship to explore either.

Since the pair-plot didn’t reveal any interesting patterns, it might be useful to look for patterns in places that didn’t show up in the pair-plot.

Looking at the dataset as a time series will show if there are any temporal patterns. An interesting variable might be number of orders over time that can give insights into user ordering patterns.

Number of orders is an important variable because:
- It coincides with delivery company’s business interests in ways described in the introduction.
- Causal relationships between time and number of orders are understandable and interpretable. Same cannot be said about for example weather or user location.

Let’s plot a heat-map with days of the week, times of the day and number of orders. This will reveal the hourly ordering patterns in our dataset.

![ordering_patterns](../../images/wolt_sarimax/ordering_patterns_heatmap.png)

**Observations**

The heat-map reveals a set of interesting patterns.

- Each day seems to have two peaks in number of orders.
- Hottest ordering times are slightly different for workdays and weekends.
- During the workdays number of orders peaks at 8am and 16pm with decrease in orders during the lunchtime.
- The weekends exhibit a similar behaviour but higher overall number of orders and with different peaks at 10–11am and 15–16pm.


![orders_over_time](../../images/wolt_sarimax/orders_over_time.png)

![orders_seasonal_decompose](../../images/wolt_sarimax/orders_seasonal_decompose.png)

**Observations**
- **Trend:** There doesn’t seem to be definite upwards or downwards trend. However, we remember from daily plot that there is a trend.
- **Seasonality:** There is very strong daily seasonal pattern. And we remember from daily plot that there is also a weekly seasonallity.
- **Residuals**: No observable patterns left in the residuals.

The strong daily seasonality in the series is a pattern worth exploring because it also coincides with business interests defined in the introduction.

Let’s analyse the seasonal pattern more in detail.

## Stationary Check
Before we try to apply any models let’s check if the time series are Stationary. Stationary comes in many flavours but here we will use the following definition: A time series is stationary if a shift in time doesn’t cause a change in the shape of its distribution. As a result of this the mean, standard deviation are not time dependent.

Fluctuating rolling mean and standard deviation can be first indication of Non-stationary time series.

![rolling_mean_std](../../images/wolt_sarimax/rolling_mean_std.png)

**Observations**
- We can see that the mean and the variance of time series are not constant over time.
- Mean and the variance seem to follow weekly seasons.

Judging from. the plot series do not look stationary.
With this in mind lets perform two statistical tests to discover if series have unit root or if they are trend-stationary.

```
Augmented Dickey Fuller Test
---------------------------------------------
ADF Statistic: -3.464712
p-value: 0.008941
Number of lags used: 24
Number of observations used: 1430
T values corresponding to adfuller test:
1% -3.434931172941245
5% -2.8635632730206857
10% -2.567847177857108

Kwiatkowski-Phillips-Schmidt-Shin test
---------------------------------------------
KPSS Statistic: 0.396855
p-value: 0.078511
Number of lags used: 20
Critical values of KPSS test:
10% 0.347
5% 0.463
2.5% 0.574
1% 0.739
```
### ADF & KPSS Test Results

- Since ADF Statistic -3.46 < -3.43 and p-value: 0.0089 < 0.05 we can reject the N0 hypothesis in the favour of Na
- Since KPSS Statistic 0.396 < 0.463 and 0.078 > 0.05 we fail to reject the N0 hypothesis.

Based on these results we can conclude that:

- According to ADF test our series have no unit root
- According to KPSS test our series are trend-stationary
- This confirms our observation from the graph above where rolling mean and std are following a weekly trend.

## Modeling with SARIMAX

Now that we established that series are trend-stationary we can start modelling.
Considering relatively small sample size, the fact that the dataset captures a seasonal time-series and the number of variables under examination, SARIMAX seems like an adequate model for the task.(see more details in code)
If a simple model can predict the target variable well, then its prediction will depend less on variables thus it will be a more general model
Therefore, for now we will not look for more complicated models that might be a better fit.

In order to choose the right SARIMAX hyperparameters let’s plot the Autocorrelation and partial autocorrelation function plots.

### ACF & PACF
ACF and PACF will help us to identify the lags that have high correlations.

![acf_pacf](../../images/wolt_sarimax/acf_pacf.png)

**Observations:**
#### ACF

- As we already knew our series are seasonal and our ACF plot confirms this pattern. If we plot more lags we will also observe that significance of the lags is gradually declining.
- First significant lag is lag 1. Which is not surprising. The number of daily orders raises/decreases gradually from hour to hour. Hence, the orders during the previous hour might tell us something about orders during the current hour.
- Netx important lags are at lag 12 and 24. These are deterministic seasonal patterns connected with day/night cycles. 12-hour lag is negatively correlated because when at 8:00am number of orders starts to increase at 20:00pm the number of orders is already decreasing. However, 24 hour lag shows that number of orders made today at 16:00pm might hint about the number of orders to be made tomorrow at 16:00pm.


#### PACF

- With PACF we can see that lag 1 and 24 have the highest correlation. This means that seasons 24 hours apart are directly correlated regardless of what is happening in between.

We can take a look at Lags of interest more in detail:

![lag_plots](../../images/wolt_sarimax/lag_plots.png)

**Observations:**

With lags 1, 12 and 24 we confirm the correlations shown in ACF plot.
Positive linear correlations can be seen in lag 1 and lag 24 and a negative non-linear correlation in lag 12.

Armed with this information lets start forging the model!

## SARIMAX
The SARIMA model is specified:

SARIMAX(p,d,q)×(P,D,Q)s

Where:

Trend Elements are:

p: Autoregressive order
d: Difference order
q: Moving average order
Seasonal Elements are:

P: Seasonal autoregressive order.
D: Seasonal difference order. D=1 would calculate a first order seasonal difference
Q: Seasonal moving average order. Q=1 would use a first order errors in the model
s Single seasonal period

Exogenous variables

X: we create an exogenous variable that would simulate the rising number of orders throughout the week.
To do this accurately we can measure the average number of orders by weekday and create a feature that can representing the day of the week.

Parameter Estimation
We will use Box–Jenkins method for identifying parameters for SARIMA.
There are of course more modern approaches like grid-search and auto.arima() but the reason we use Box–Jenkins is that it forces you to understand your model as opposed to brute forcing your way through the parameters.
First we will try to estimate the parameters based on our theoretical understanding of ACF and PACF plots to ensure that we understand our model.
Next, we will cross-check our values with the results of grid search.

**Theoretical estimates:**

s: In our ACF plot there is one peak and one valley every 24 hours. Thus, we can set seasonal period to s = 24. This also backed by our subject matter knowledge.

p: We are dealing with a gradual change where yt−1 is not drastically different from yt hence the trend autoregressive order will be set to p = 1. This is also confirmed in the ACF plot where yt−1 is the first significant lag.

d: We established that our series are trend-stationary we will set trend differencing to d = 1

q: Based on our PACF correlations we can set q = 2 since its the most significant lag.

P: P = 2 will allow us to use the first and second seasonally offsets (24) in the model, e.g. t−(s×1)=t−(24×1), t−(s×1)=t−(24×2)

D: Since we are dealing with seasonality we can use first degree seasonal differencing D = 1

Q: The seasonal moving average will be set to Q = 2. In this case the model will take into account the moving average of lag t−(24×1) and t−(24×2) as shown in our PACF graph these lags have a significant correlation.

Resulting model is:
SARIMA(1,1,1)×(2,1,1)24

### Training
Now let’s train the model! 🚀

```
SARIMAX Results                                      
==========================================================================================
Dep. Variable:                             ORDERS   No. Observations:                 1075
Model:             SARIMAX(1, 1, 2)x(2, 1, 2, 24)   Log Likelihood               -3118.452
Date:                            Mon, 07 Feb 2022   AIC                           6254.904
Time:                                    15:31:00   BIC                           6299.513
Sample:                                08-01-2020   HQIC                          6271.818
                                     - 09-15-2020                                         
Covariance Type:                              opg                                         
================================================================================
                   coef    std err          z      P>|z|      [0.025      0.975]
--------------------------------------------------------------------------------
weekday_exog     0.8400      0.138      6.073      0.000       0.569       1.111
ar.L1            0.4937      0.063      7.783      0.000       0.369       0.618
ma.L1           -1.1574      0.079    -14.569      0.000      -1.313      -1.002
ma.L2            0.1579      0.070      2.266      0.023       0.021       0.295
ar.S.L24         0.8870      0.052     17.011      0.000       0.785       0.989
ar.S.L48        -0.2624      0.024    -10.926      0.000      -0.309      -0.215
ma.S.L24        -1.7488      0.052    -33.725      0.000      -1.850      -1.647
ma.S.L48         0.7709      0.051     15.025      0.000       0.670       0.871
sigma2          20.6057      0.886     23.252      0.000      18.869      22.343
===================================================================================
Ljung-Box (L1) (Q):                   0.00   Jarque-Bera (JB):               339.23
Prob(Q):                              0.98   Prob(JB):                         0.00
Heteroskedasticity (H):               1.34   Skew:                             0.54
Prob(H) (two-sided):                  0.01   Kurtosis:                         5.57
===================================================================================
```
![train_eval](../../images/wolt_sarimax/train_eval.png)

**Observations:**

- The standardized residual plot: The residuals over time don’t display any obvious patterns. They appear as white noise.
- The Normal Q-Q-plot: Shows that the ordered distribution of residuals follows the linear trend. However, the curving in the plot suggest heavier tails in our distribution.
- Histogram and estimated density plot: The KDE follows the N(0,1) distribution however it has a positive kurtosis.
- The Correlogram plot: Shows that the time series residuals have low correlation with lagged versions of itself. Thus, there are no significant patterns left to extract in the residuals.


### Evaluate

![eval](../../images/wolt_sarimax/eval.png)

```
Akaike information criterion | AIC: 6254.90413501809
Bayesian information criterion | BIC: 6299.513044006454
Mean Squared Error | MSE: 22.106947769139186
Sum Squared Error | SSE: 23764.968851824626
Root Mean Squared Error | RMSE: 5.132799959869062
```

The key metric to pay attention to here is:

```
Root Mean Squared Error | RMSE: 5.132799959869062
```
Root Mean Squared Error shows us that on average our model makes a mistake in predictions of 5 orders. Depending on our goals we can optimise the model for a smaller error however for the purposes of this article this is an acceptable result.
We can proceed to the forecasting.

![forecast](../../images/wolt_sarimax/forecast.png)

## Conclusion

In this notebook we explored the provided dataset and implemented a SARIMAX model. The model captured the two daily spikes but not when these spikes crossed the mark of 40 orders. It accurately replicated the gradual rise of orders throughout the week. However, the model predicts orders during the night hours that are very unlikely to occur. It seems that the model would benefit from another Exog variable that would apply hourly weights for each hour of the day. We can see that our model minimised RMSE down to 5 orders.

Further development

There are multiple directions we can move from here. If we stick with current SARIMAX model we could:

- Account for public holidays and other anomalies.
- Incorporate monthly and seasonal(summer/winder) trends.
- Data can be spatialized and the models can be applied on a more granular level i.e. neighbourhood, restaurant, user category.
- 
Naturally all will require more data and features.
However, if new data and features are added they should be in some form of interpretable causal relationships with the target variable.
We had more data in our sample dataset, but it did not seem to be directly applicable because establishing the causal relationships between these variables would not have been possible.

Alternatively we can try to apply different models.
One candidate could be Exponential smoothing, although it will probably struggle with multi seasonal dataset like this.
If we had more data available we could apply Propet, XGboost or even dive into the rabbit hole of neural networks.
But these topics are a discussion another time :)

Full Jupyter notebook with the code can be found in this repo: [![GitHub](https://img.shields.io/badge/GitHub-%23121011.svg?logo=github&logoColor=white)](https://github.com/Geometrein/sarimax)


