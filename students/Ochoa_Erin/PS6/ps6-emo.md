MACS 30100 PS6
================
Erin M. Ochoa
2017 February 20

-   [Part 1: Modeling voter turnout](#part-1-modeling-voter-turnout)
    -   [Describing the data](#describing-the-data)
    -   [Estimating a basic logistic model](#estimating-a-basic-logistic-model)
    -   [Multiple variable model](#multiple-variable-model)
-   [Part 2: Modeling television consumption](#part-2-modeling-television-consumption)
    -   [Estimate a regression model (3 points)](#estimate-a-regression-model-3-points)
-   [Submission instructions](#submission-instructions)
    -   [If you use R](#if-you-use-r)

Part 1: Modeling voter turnout
==============================

The 1998 General Social Survey included several questions about the respondent's mental health. `mental_health.csv` reports several important variables from this survey.

-   `vote96` - 1 if the respondent voted in the 1996 presidential election, 0 otherwise
-   `mhealth_sum` - index variable which assesses the respondent's mental health, ranging from 0 (an individual with no depressed mood) to 9 (an individual with the most severe depressed mood)[1]
-   `age` - age of the respondent
-   `educ` - Number of years of formal education completed by the respondent
-   `black` - 1 if the respondent is black, 0 otherwise
-   `female` - 1 if the respondent is female, 0 if male
-   `married` - 1 if the respondent is currently married, 0 otherwise
-   `inc10` - Family income, in $10,000s

We begin by reading in the data. Because it does not make sense to consider respondents for whom either voting behavior or depression index score is missing, we subset the data to include only those respondents with valid responses in both variables. We also add a factor-type variable to describe voting; this will decrease the time necessary to construct plots.

``` r
df = read.csv('data/mental_health.csv')
df = df[(!is.na(df$vote96) & !is.na(df$mhealth_sum)), ]
df$Turnout = factor(df$vote96, labels = c("Did not vote", "Voted"))
```

Describing the data
-------------------

1.  We plot a histogram of voter turnout:

![](ps6-emo_files/figure-markdown_github/vote_histogram-1.png)

The unconditional probability of a given respondent voting in the election is 0.672466. The data are distributed bimodally, with about twice as many respondents voting as not voting.

1.  We generate a scatterplot of voter turnout versus depression index score, with points colored by whether the respondent voted. Because mental health scores are integers ranging from \[0,16\] and turnout is categorical, there can be a maximum of 34 points on the plot. This would not be terribly informative, so we jitter the points, increase their transparency, and add a horizontal line between the distributions of voters and non-voters; however, we must remember that the jittered position is not the true position: we must imagine the same number of points, with all the blue ones at 1 and all the red ones at 0. (That is: any within-group variability in the y direction is false.) These additions are somewhat helpful, but because voter turnout is dichotomous, it is not well suited to a scatterplot. (We will address this soon.)

![](ps6-emo_files/figure-markdown_github/scatterplot_turnout_vs_mentalhealth-1.png)

The regression line shows that respondents with higher depression scores trend toward not voting. We note again, however, that because voter turnout is dichotomous—a respondent either votes (1) or doesn't (0), with no possile outcomes in between—the regression line is misleading. It suggests, for example, that potential respondents with scores so high that they are off the index could have a negative chance of voting, which makes no sense; similarly, respondents with scores well below zero could have greater than a 1.0 chance of voting. Additionally, because the depression index score ranges from \[0,16\], it does not have support over the entire domain of real numbers; the regression line, however, suggests that such scores are possible and points to probabilites for them—and some of those probabilites fall outside of the real of possible prbabilities (which range from \[0,1\]). These problems imply that linear regression is the wrong type of analysis for the type of data with which we are dealing.

We now return to the matter of visualizing the distribution of depression scores by voter turnout. Because the outcome is dichotomous and the predictor is continuous but over a short interval, the scatterplot does a poor job of clearly showing the correlation between depression score and turnout. We therefore turn to a density plot:

![](ps6-emo_files/figure-markdown_github/density_plot-1.png)

Now we can clearly see that voters and non-voters had very different distributions of depression scores: most voters scored between \[0,5\] on the depression index, but only about half of non-voters did. While there were voters and non-voters over nearly the entire range of possible depression scores, non-voters tended to have higher scores (but the only respondents to score 16 on the depression index were actually voters).

Estimating a basic logistic model
---------------------------------

First, we define the functions necessary in this section:

``` r
logit2prob = function(x){
  exp(x) / (1 + exp(x))
}

prob2odds = function(x){
  x / (1 - x)
}

prob2logodds = function(x){
  log(prob2odds(x))
}

calcodds = function(x){
  exp(int + coeff * x)
}

oddsratio = function(x,y){
  exp(coeff * (x - y))
}

calcprob = function(x){
  exp(int + coeff * x) / (1 + exp(int + coeff * x))
}

firstdifference = function(x,y){
  calcprob(y) - calcprob(x)
}

threshold_compare = function(thresh, dataframe, model){
  pred <- dataframe %>%
          add_predictions(model) %>%
          mutate(pred = logit2prob(pred),
          pred = as.numeric(pred > thresh))
  
  cm = confusionMatrix(pred$pred, pred$vote96, dnn = c("Prediction", "Actual"), positive = '1')

  data_frame(threshold = thresh,
             sensitivity = cm$byClass[["Sensitivity"]],
             specificity = cm$byClass[["Specificity"]],
             accuracy = cm$overall[["Accuracy"]])
}
```

We estimate a logistic regression model of the relationship between mental health and voter turnout:

``` r
logit_voted_depression = glm(vote96 ~ mhealth_sum, family = binomial, data=df)
summary(logit_voted_depression)
```

    ## 
    ## Call:
    ## glm(formula = vote96 ~ mhealth_sum, family = binomial, data = df)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -1.6834  -1.2977   0.7452   0.8428   1.6911  
    ## 
    ## Coefficients:
    ##             Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept)  1.13921    0.08444  13.491  < 2e-16 ***
    ## mhealth_sum -0.14348    0.01969  -7.289 3.13e-13 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 1672.1  on 1321  degrees of freedom
    ## Residual deviance: 1616.7  on 1320  degrees of freedom
    ## AIC: 1620.7
    ## 
    ## Number of Fisher Scoring iterations: 4

We generate the additional dataframes and variables necessary to continue:

``` r
int = tidy(logit_voted_depression)[1,2]
coeff = tidy(logit_voted_depression)[2,2]

voted_depression_pred = df %>%
                        add_predictions(logit_voted_depression) %>%
                        mutate(prob = logit2prob(pred)) %>%
                        mutate(odds = prob2odds(prob))

voted_depression_accuracy = df %>%
                            add_predictions(logit_voted_depression) %>%
                            mutate(pred = logit2prob(pred),
                            pred = as.numeric(pred > .5))
```

1.  We find a statistically significant relationship at the p &lt; .001 level between depression score and voting behavior. The relationship is negative and the coefficient is -0.1434752. Because the model is not linear, we cannot simply say that a change in depression index score results in a corresponding change in voter turnout. Instead, we must interpret the coefficient in terms of log-odds, odds, and probability. We will interpret the coefficient thus in the following responses.

2.  Log-odds: For every one-unit increase in depression score, we expect the log-odds of voting to decrease by -0.1434752.

We graph the relationship between mental health and the log-odds of voter turnout:

![](ps6-emo_files/figure-markdown_github/log_odds_plot-1.png)

1.  Odds: The coefficient for depression index score cannot be interpreted in terms of odds without being evaluated at a certain depression score. This is because the relationship between depression score and odds is logistic, not linear:

$$
Odds\\\_of\\\_voting = \\frac{p(depression\\\_index\\\_score)}{1 - p(depression\\\_index\\\_score)} = e^{1.13921 - (0.1434752 \\times depression\\\_index\\\_score)}
$$
 For example, for a respondent with a depression index score of 12 would have odds of voting equal to 0.5585044. This means the respondent is 0.5585044 times more likely to vote than not vote, because this is less than one, such a respondent would be unlikely to vote. In contrast, a respondent with a depression index score of 3 would be 2.0315196 times more likely to vote than not vote. A respondent with a depression score of 8 would be approximately just as likely to vote as not vote because the odds for that score equal 0.9914448.

We graph the relationship between depression index score and the odds of voting:

![](ps6-emo_files/figure-markdown_github/voted_depression_odds_plot-1.png)

1.  Probability: The relationship between depression index score and voting is not linear; like with odds, we must use a specific depression index score in order to calculate the probability of such a respondent voting. For example, a respondent with a depression index score of 3 would have a probability of voting equal to 0.6701324 and a respondent who scored 12 would have a probability of 0.3583592. As we noted earlier, a respondent with a score of 8 would be about equally likely to vote as not vote, with a probability of 0.497852.

The first difference for an increase in the mental health index from 1 to 2 is -0.0291782; for 5 to 6, it is -0.0347782.

We plot the probabilty of voting against depression score, including the actual responses as points (this time, without jitter):

![](ps6-emo_files/figure-markdown_github/logit_voted_depression_prob_plot-1.png)

We define the variables necessary to answer the next question:

``` r
ar = mean(voted_depression_accuracy$vote96 == voted_depression_accuracy$pred, na.rm = TRUE)

uc = median(df$vote96)

e1 = sum(df$vote96 != uc)
e2 = sum(voted_depression_accuracy$pred != uc, na.rm = TRUE)

pre = (e1 - e2) / e1

cm.5_voted_depression <- confusionMatrix(voted_depression_accuracy$pred, voted_depression_accuracy$vote96,
                         dnn = c("Prediction", "Actual"), positive = '1')

cm.5_table = cm.5_voted_depression$table


actlpos = cm.5_table[1,2] + cm.5_table[2,2]
predposcrrct = cm.5_table[2,2]

actlneg = cm.5_table[1,1] + cm.5_table[2,1]
prednegcrrct = cm.5_table[1,1]

tpr.notes =  predposcrrct / actlpos
tnr.notes =  prednegcrrct / actlneg

tpr.cm.5 = sum(cm.5_voted_depression$byClass[1])
tnr.cm.5 = sum(cm.5_voted_depression$byClass[2])


threshold_x = seq(0, 1, by = .001) %>%
              map_df(threshold_compare, df, logit_voted_depression)

auc_x_voted_depression <- auc(df$vote96, voted_depression_pred$prob)
```

1.  Using a cutoff of .5, we estimate the accuracy rate of the model at 0.677761.

We find that the useless classifier for this data predicts that all voters will vote; because the voter variable is dichotomous, we find this by simply taking the median of the distribution: 1.

With the useless classifier (which predicts all respondents will vote), we find that the proportional reduction in error is 0.7344111. This means that the model based only on depression index scores provides an improvement in the proportional reduction in error of 73.4411085 over the useless-classifier model.

The AUC score for this model is 0.6243087.

This model performs reasonably well, especially considering that it uses only one predictor. With a moderately high proportional reduction in error based on the 50% threshold as well as moderate accuracy rate and AUC, the model performs surprisingly well given the single predictor, depression index score, on which it is based. We expect to improve the model by including additional predictors.

For good measure we plot the accuracy, sensitivity, and specificy rates for thresholds between 0 and 1:

![](ps6-emo_files/figure-markdown_github/ar_vs_threshold_plot-1.png)

We also plot the ROC curve:

![](ps6-emo_files/figure-markdown_github/roc_plot-1.png)

Multiple variable model
-----------------------

Using the other variables in the dataset, derive and estimate a multiple variable logistic regression model of voter turnout.

1.  Write out the three components of the GLM for your specific model of interest. This includes the
    -   Probability distribution (random component): Because we are using logistic regression, we assume that the outcome (voted or did not vote) is drawn from the Bernoulli distribution, with probability *π*:

*P**r*(*Y*<sub>*i*</sub> = *y*<sub>*i*</sub>|*π*<sub>*i*</sub>)=*π*<sub>*i*</sub><sup>*y*<sub>*i*</sub></sup>(1 − *π*<sub>*i*</sub>)<sup>(1 − *y*<sub>*i*</sub>)</sup>

    * Linear predictor: The linear predictor is the following multivariate linear model:

*g*(*π*<sub>*i*</sub>)≡*η*<sub>*i*</sub> = *β*<sub>0</sub> + *β*<sub>1</sub>*D**e**p**r**e**s**s**i**o**n**I**n**d**e**x**S**c**o**r**e*<sub>*i*</sub> + *β*2*A**g**e*<sub>*i*</sub> + *β*<sub>3</sub>*I**n**c**o**m**e*10*K*<sub>*i*</sub>
 \* Link function: The link function is the logit function:

$$
\\pi\_i = \\frac{e^{\\eta\_i}}{1 + e^{\\eta\_i}}
$$

1.  We estimate the model:

``` r
#logit_voted_mv = glm(vote96 ~ age + inc10, family = binomial, data=df)
logit_voted_mv = glm(vote96 ~ mhealth_sum + age + inc10, family = binomial, data=df)

summary(logit_voted_mv)
```

    ## 
    ## Call:
    ## glm(formula = vote96 ~ mhealth_sum + age + inc10, family = binomial, 
    ##     data = df)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.2470  -1.1170   0.6101   0.8653   1.7640  
    ## 
    ## Coefficients:
    ##              Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept) -0.953205   0.238966  -3.989 6.64e-05 ***
    ## mhealth_sum -0.107597   0.022582  -4.765 1.89e-06 ***
    ## age          0.032162   0.004355   7.384 1.53e-13 ***
    ## inc10        0.143886   0.023571   6.104 1.03e-09 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 1472.2  on 1167  degrees of freedom
    ## Residual deviance: 1314.5  on 1164  degrees of freedom
    ##   (154 observations deleted due to missingness)
    ## AIC: 1322.5
    ## 
    ## Number of Fisher Scoring iterations: 4

We generate the additional dataframes and variables necessary to answer the question:

``` r
int_mv = tidy(logit_voted_mv)[1,2]
coeff_mv_mh = tidy(logit_voted_mv)[2,2]
coeff_mv_age = tidy(logit_voted_mv)[3,2]
coeff_mv_inc = tidy(logit_voted_mv)[4,2]

voted_mv_pred = df[(!is.na(df$age) & !is.na(df$inc10)), ] %>%
                #data_grid(mhealth_sum, age, inc10) %>%
                add_predictions(logit_voted_mv) %>%
                mutate(prob = logit2prob(pred)) %>%
                mutate(odds = prob2odds(prob))

med_age = median(voted_mv_pred$age)
med_inc = median(voted_mv_pred$inc10)

attach(voted_mv_pred)

voted_mv_pred$age_inc = 0
voted_mv_pred$age_inc[age < med_age & inc10 >= med_inc] = 1
voted_mv_pred$age_inc[age < med_age & inc10 < med_inc] = 2
voted_mv_pred$age_inc[age >= med_age & inc10 >= med_inc] = 3
voted_mv_pred$age_inc[age >= med_age & inc10 < med_inc] = 4

voted_mv_pred$age_inc = factor(voted_mv_pred$age_inc, labels = c("Younger, higher income", "Younger, lower income", "Older, higher income", "Older, lower income"))
  
  
voted_mv_accuracy <- df[(!is.na(df$age) & !is.na(df$inc10)), ] %>%
                     add_predictions(logit_voted_mv) %>%
                     mutate(pred = logit2prob(pred),
                     pred = as.numeric(pred > .5))
```

1.  Interpret the results in paragraph format. This should include a discussion of your results as if you were reviewing them with fellow computational social scientists. Discuss the results using any or all of log-odds, odds, predicted probabilities, and first differences - choose what makes sense to you and provides the most value to the reader. Use graphs and tables as necessary to support your conclusions.

We find that depression index score, age, and education are statistically significant at the p&lt;.001 level and income (in tens of thousands) is significant at the p&lt;.01 level.

![](ps6-emo_files/figure-markdown_github/mv_log_odds_plot-1.png)

![](ps6-emo_files/figure-markdown_github/voted_mv_odds_plot-1.png)

![](ps6-emo_files/figure-markdown_github/logit_mv_prob_plot-1.png)

Part 2: Modeling television consumption
=======================================

We begin by reading in the data. Having chosen the variables of interest, we subset the dataframe such that all cases contain valid responses for all such variables. We also convert social\_connect to a factor variable and label the levels.

``` r
df2 = read.csv('data/gss2006.csv')
df2 = df2[(!is.na(df2$tvhours) & !is.na(df2$hrsrelax) & !is.na(df2$social_connect)), ]
df2$social_connect = factor(df2$social_connect, labels = c("Low", "Medium", "High"))
```

In this part of the problem set, you are going to derive and estimate a series of models to explain and predict TV consumption, or the number of hours of TV watched per day. As this is an event count, you will use Poisson regression to model the response variable. `gss2006.csv` contains a subset of the 2006 survey which contains many variables you can use to construct a model.

-   `tvhours` - The number of hours of TV watched per day
-   `age` - Age (in years)
-   `childs` - Number of children
-   `educ` - Highest year of formal schooling completed
-   `female` - 1 if female, 0 if male
-   `grass` - 1 if respondent thinks marijuana should be legalized, 0 otherwise
-   `hrsrelax` - Hours per day respondent has to relax
-   `black` - 1 if respondent is black, 0 otherwise
-   `social_connect` - Ordinal scale of social connectedness, with values low-moderate-high (0-1-2)
-   `voted04` - 1 if respondent voted in the 2004 presidential election, 0 otherwise
-   `xmovie` - 1 if respondent saw an X-rated movie in the last year, 0 otherwise
-   `zodiac` - Respondent's [astrological sign](https://en.wikipedia.org/wiki/Astrological_sign)

Estimate a regression model (3 points)
--------------------------------------

Using the other variables in the dataset, derive and estimate a multiple variable Poisson regression model of hours of TV watched.

1.  Write out the three components of the GLM for your specific model of interest. This includes the
    -   Probability distribution (random component)
    -   Linear predictor
    -   Link function

2.  Estimate the model and report your results.

We estimate a Poisson model to explain television consumption with leisure time and social connectedness:

``` r
poisson_tv <- glm(tvhours ~ hrsrelax + social_connect, family = "quasipoisson", data = df2)
summary(poisson_tv)
```

    ## 
    ## Call:
    ## glm(formula = tvhours ~ hrsrelax + social_connect, family = "quasipoisson", 
    ##     data = df2)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.8981  -0.9040  -0.2332   0.4077   6.4822  
    ## 
    ## Coefficients:
    ##                      Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)          0.702974   0.044485  15.803  < 2e-16 ***
    ## hrsrelax             0.041523   0.007125   5.828 7.34e-09 ***
    ## social_connectMedium 0.067613   0.051549   1.312     0.19    
    ## social_connectHigh   0.066088   0.056189   1.176     0.24    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for quasipoisson family taken to be 1.347905)
    ## 
    ##     Null deviance: 1346.9  on 1119  degrees of freedom
    ## Residual deviance: 1301.0  on 1116  degrees of freedom
    ## AIC: NA
    ## 
    ## Number of Fisher Scoring iterations: 5

``` r
df2 %>%
  data_grid(hrsrelax, social_connect) %>%
  add_predictions(poisson_tv) %>%
  ggplot(aes(hrsrelax, pred, color=social_connect)) +
  geom_line(size = 1.5) +
  labs(x = "Hours of Relaxation",
       y = "Predicted log-count of television hours")
```

![](ps6-emo_files/figure-markdown_github/poisson_log_count_plot-1.png)

``` r
df2 %>%
  data_grid(hrsrelax, social_connect) %>%
  add_predictions(poisson_tv) %>%
  mutate(pred = exp(pred)) %>%
  ggplot(aes(hrsrelax, pred, color=social_connect)) +
  geom_line(size=1.5) +
  labs(x = "Hours of Relaxation",
       y = "Predicted count of television hours")
```

![](ps6-emo_files/figure-markdown_github/poisson_tv_predicted_count_plot-1.png)

1.  Interpret the results in paragraph format. This should include a discussion of your results as if you were reviewing them with fellow computational social scientists. Discuss the results using any or all of log-counts, predicted event counts, and first differences - choose what makes sense to you and provides the most value to the reader. Is the model over or under-dispersed? Use graphs and tables as necessary to support your conclusions.

Submission instructions
=======================

Assignment submission will work the same as earlier assignments. Submit your work as a pull request before the start of class on Monday. Store it in the same locations as you've been using. However the format of your submission should follow the procedures outlined below.

If you use R
------------

Submit your assignment as a single [R Markdown document](http://rmarkdown.rstudio.com/). R Markdown is similar to Juptyer Notebooks and compiles all your code, output, and written analysis in a single reproducible file.

[1] The variable is an index which combines responses to four different questions: "In the past 30 days, how often did you feel: 1) so sad nothing could cheer you up, 2) hopeless, 3) that everything was an effort, and 4) worthless?" Valid responses are none of the time, a little of the time, some of the time, most of the time, and all of the time.