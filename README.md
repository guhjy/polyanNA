# polyanNA

("Polynomial Analysis with NAs")

Novel, **nonimputational**  methods for handling missing values (MVs) in
prediction applications.

## Overview

The intended class of applications is regression modeling, at this time
linear and generalized linear models (nonparametric/ML models to be
included later).  

Unlike most of the MV literature, our emphasis is on prediction, rather
than on estimation of regression coefficients and the like.  This is
important first because we are in the era of Big Data, which is
prediction-oriented, but just as importantly, also because of the issue
of assumptions.  Most MV methods (including ours) make strong
assumptions, which are difficult or impossible to verify.  We posit that
the prediction context is more robust to the assumptions than is
estimation.  This would be similar to the non-MV setting, in which
models can be rather questionable yet still have strong predictive
power.

We are especially interested in predicting new cases that have missing
values. In such settings, the classic Complete Cases method (CCM) --
use only fully intact rows -- is useless.

To make things concrete, say we are regressing Y on a vector X of length
p.  We have data in a matrix A of n rows, thus of dimensions n X p.
Some of the elements of A are missing, i.e. are NA values in the R
language.

Note carefully that in describing our methods as being for regression
applications, *we do NOT mean imputing missing values through some
regression technique.* Instead, our context is that of regression
applications themselves.  Again, all of our methods are
**nonimputational**.  (For some other nonimputational methods, see for
instance (Soysal, 2018) and (Matloff, 2015).)

## Contributions of This Package

1. Cross-product extensions of the Missing-Indicator Method.

2. A novel approach, based on the Tower Property of expected values.

## Extending the Missing-Indicator Method

Our first method is an extension of the Missing-Indicator Method
(Miettinen, 1983) (Jones 1996).  MIM can be described as follows.

*MIM method*

Say X includes some numeric variable, say Age. MIM add a new column to
A, say Age.NA, consisting of 1s and 0s.  Age.NA is a missingness
indicator for Age:  For any row in A for which Age is NA, the NA is
replaced by 0, and Age.NA is set to 1; otherwise Age.NA is 0.  So rather
than trying to get the missing value, we treat the missingness as
informative, with the information being carried in Age.NA.

It is clear that replacement of an NA by a 0 value -- fake and likely highly
inaccurate data -- can induce substantial bias, say in the regression
coefficient of Age.  Indeed, some authors have dismissed MIM as only
useful back in the era before modern, fast computers that 
can handle computationally intensive imputational methods
(Nur, 2010). 

*Case of categorical variables*

However, now consider categorical variables, i.e. R factors.  Here,
instead of adding a fake 0, we in essence merely add a legitimate new
level to the factor (Jones, 1996).

Say we have a predictor variable EyeColor, taking on values Brown, Blue,
Hazel and Green, thus an R factor with these four levels.  In, say,
**lm()**, R will convert this one factor to three dummy variables, say
EyeColor.Brown, EyeColor.Blue and EyeColor.Hazel.  MIM then would add a
dummy variable for missingness, EyeColor.na, and in rows having a
value of 1 for this variable, the other three dummies would be set to 0, 
We then proceed with our regression analysis as usual, using the modified
EyeColor factor/dummies.

Unlike the case of numeric variables, this doesn't use fake data, and
produces no distortion.  

**NOTE:** Below, all mentions of MIM refer specifically to that method
as applied to categorical variables.
 
*Value of MIM*

As with imputational methods, the non-imputational MIM is motivated by a
desire to make use of rows of A having non-missing values, rather than
discarding them as in CCM.  Note again that this is crucial in our
prediction context; CCM simply won't work here.

Moreover, MIM treats NA values as potentially *informative*; ignoring
them may induce a bias.  MIM may reduce this bias, by indirectly
accounting for the distributional interactions between missingness and
other variables.  This occurs, for instance, in the cross-product terms
in A'A and A'B, where the **lm()** coefficient vector is (A'A)<sup>-1</sup>
A'B.


*The polyanNA package*

The first method offered in our **polyanNA**  package implements MIM for
factors, both in its traditional form, and with polynomial extensions to be
described below.

The **mimPrep()** function inputs a data frame and  converts all factor
columns according to MIM.  Optionally, the function will discretize the
numeric columns as well, so that they too can be "MIM-ized."

There is also a function **lm.pa()**, with an associated method for the
generic **predict()**, to implement linear modeling and prediction in
MIM settings, including polynomial MIM.

*Example* 

The function **lm.pa.ex1()**, included in the package, inputs some Census
data on programmer and engineer wages in 2000.  It regresses WageIncome
against Age, Education, Occupation, Gender, and WeeksWorked.

We intentionally inject NA values in the Occupation variable,
specifically in about 10% of the cases in which Occupation has code 102,
one of the higher-paying categories:  

```
> tapply(pe$wageinc,pe$occ,mean)
     100      101      102      106      140      141 
50396.47 51373.53 68797.72 53639.86 67019.26 69494.44 
```

In this (artificial) setting, the missingness of Occupation tells us
that the actual value 102 or 104.

This experiment is interesting because the proportion of women in those
two occupations is low:

```
> table(pe[,c(5,7)])
     sex
occ      0    1
  100 1530 3062
  101 1153 3345
  102 1607 5213
  106  209  292
  140  127  675
  141  282 2595
```

Due to the fact that there are fewer women in occupations 102 and 140, two
high-paying occupations, a naive regression analysis using CCM might bias
the gender effect downward.  Let's see.

Though we are primarily interested in prediction, let's look at the
estimated coefficient for Gender.

<pre>
Run    full           CC         MIM
1      8558.765       8391.369   8581.544 
2      8558.765       8358.057   8438.852 
3      8558.765       8717.961   8544.091
4      8558.765       8573.875   8585/638
5      8558.765       8270.693   8473.888
</pre>

These numbers are very gratifying. We see that CCM produces a bias, but
that the bias is ameliorated by MIM.

*Extension using a polynomial model*

Now, note that if single NA values are informative, then pairs or
triplets of NAs and so on may carry further information.  In other
words, we should consider forming products of the dummy variables,
between one of the original factors and another.

We handle this by using our [polyreg
package](http://github/matloff/polyreg), which forms polynomial terms in
a *multivariate* context, properly handling the case of indicator
variables. It accounts for the fact that powers of dummies need not be
computed, and that products of dummy columns from the same categorical
variable will be 0.

*MIM functions in this package*

**mimPrep(xy,yCol=NULL,breaks=NULL,allCodeInfo=NULL)**

Applies MIM to all columns in **xy** that are factors, other than a Y
column if present (non-NULL **yCol**, with the value indicating the
column number of Y).  Optionally first discretizes all numeric columns
(other than Y), setting breaks levels.  

Used both on training data and later in prediction.  In the former case,
**allCodeInfo** is NULL, but in the latter case, after fitting, say,
**lm()** to the training data, one saves the value of **allCodeInfo** found
by **mimPrep()** on that data.  Then in predicting new cases, one sets
**allCodeInfo** to that saved value.  In this manner, we ensure that the
same MIM operations are used both in training and later prediction.
 
**lm.pa(paout,maxDeg=1,maxInteractDeg=1)**  The degree of polynomial
used is specified by the remaining two arguments; see the **polyreg**
documentation for details.  [DEGREE > 1 UNDER CONSTRUCTION]

This is a wrapper for **lm()**.  The argument **paout** is the return
value from a call to **mimPrep()** 

**predict.lm.pa(lmpaout,newx)**

[UNDER CONSTRUCTION]

## toweranNA(): A novel method based on regression averaging

In an early paper (Matloff, 1981), the following result was proved: Fit
a parametric model (linear or nonlinear) to a sample of size n, then
average the fitted values, providing an estimate of EY, the population
unconditional mean of Y.  Then the asymptotic variance of this estimator
is smaller than that of the sample mean of Y, except if the model is
linear with a constant term.

That result was motivated by the famous formula

```
   EY = E[E(Y | X)]
```

A more general version is known as the Tower Property.  For random
variables Y, U and V, 

``` 
   E[ E(Y|U,V) | U ] = E(Y | U) 
``` 
   
What this theoretical abstraction says is that if we take the regression
function of Y on U and V, and average it over V for fixed U, we get the
regression function of Y on U.  

If V is missing but U is known, this is very useful.  Take the Census
data above on programmer and engineer wages, with predictors age,
education, occupation, gender and number of weeks worked. Say we wish to
predict case in which age and gender are missing.  Then (under proper
assumptions), our prediction might be the estimated value of the
regression function of wage on education, occupation and weeks worked.

Though there are only 5 predictor variables in this dataset, once the
factors are expanded to dummies, it becomes 23 predictors, with
2<sup>23</sup> possible NA patterns.  It would be impractical to fit and
verify marginal regression function models for all these patterns.  

But the Tower Property provides an alternative.  We fit the full model
to the complete cases in the data, then average that model over all data
points having education, occupation and weeks worked as in the new case
to be predicted.  In the Tower Property above, U would be these
variables, while V would be the vector (age,education).

Our function **toweranNA()** ("tower analysis with NAs") takes this
approach.  Usually, there will not be many data points having the exact
value specified for U, so we average over a neighborhood of points near
that value.  

The call form is

``` r
toweranNA(x, fittedReg, k, newx, scaleX = TRUE) 
```

where the arguments are: the data frame of the "X" data in the
training set; the vector of fitted regression function values from that
set; the number of nearest neighbors; and the "X" data frame for the
data to be predicted.

The number of neighbors is of course a tuning parameter chosen by the
analyst.  Since we are averaging fitted regression estimates, which are
by definition already smoothed, a small value of k should work well.  We
actually set **k = 1** as the default.

The function **doExpt1()** allows one to explore the behavior of the
Tower method on real data.  As an example, we again look at the Census
data, predicting wage income from age, gender, weeks worked, education
and occupationr:

``` R
getPE(Dummies=TRUE)
pe1 <- pe[,c(1:4,6,7,12:16)]
pe2 <- pe1[,c(1,2,4:11,3)]  # "Y" var last
 ```

The function **doExpt1()** randomly culls 500 observations from the
data, and inserts NAs.  Here we will set education to NA in the first
250 data points, and set occupation to NA in the rest.  We then compare
Tower and **mice**.  Below are the mean absolute prediction errors for
**k = 1**, over five runs, from calling **doExpt1(pe2,4:5,6:10)**.

```
Tower    mice
27814.86 28454.95
24440.01 24907.24
26860.15 26875.05
26455.16 26880.92
28679.61 29334.60
```

And the same for **k = 5**:

```
Tower    mice
26086.31 26929.85 
22096.91 22863.99 
26034.20 26582.75 
25409.16 26021.63 
24278.39 25021.50 
```

It is important to note that, in addition to Tower's consistently
achieving better accuracy than **mice**, Tower is also a lot faster,
about 4 seconds vs. 31.

## Assumptions

We will not precisely define assumptions underlying the above methods
here; roughly, they are similar to those most existing methods.
However, as noted, our view is that prediction contexts are more robust to
the assumptions, as seen in the examples above.

## Utility functions in this package

**factorsToDummies(df)**

Inspects each column of the data frame **df**.  If a column is numeric,
it is copies without change.  If it is a factor, it is converted to two
or more columns of dummy variables.  Return value is the expanded
version of **df**.

## References

X. Gu and N. Matloff, A Different Approach to the Problem of Missing
Data, *Proceedings of the Joint Statistical Meetings*, 2015

M. Jones, Indicator and Stratification Methods for Missing Explanatory
Variables in Multiple Linear Regression, *JASA*m 1996

O.S. Miettinen, *Theoretical Epidemiology:
Principles of Occurrence Research in Medicine*, 1985

S. Soysal *et al*, the Effects of Sample Size and Missing Data Rates on
Generalizability Coefficients, *Eurasian J. of Ed. Res.*, 2018

