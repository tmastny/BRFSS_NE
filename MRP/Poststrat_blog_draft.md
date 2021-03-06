# Weighted Averages and Poststratification
Tim  
11/17/2017  




## Introduction

This blog will lead into the next post on Multilevel Regression and Poststratification, [or MRP](http://andrewgelman.com/2013/10/09/mister-p-whats-its-secret-sauce/). Before that I'd like to go over some basics of poststratification and weighted averages. 

First, let's imagine we have the following data. 


```r
vote_yes <- c(rep(0, 25*(1-0.76)), 
              rep(1, 25*0.76),
              rep(0, 50*(1-0.3)),
              rep(1, 50*0.3))
vote_yes
```

```
##  [1] 0 0 0 0 0 0 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0
## [36] 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 1 1 1 1 1 1 1 1 1
## [71] 1 1 1 1 1
```

Suppose this data is a list of yes or no responses to a public policy poll. Based on this sample of the population, we can gauge whether there is a majority of support for an issue by taking the average. Let $X = x_1, x_2, \dots, x_{75}$ denote this data set. Then


```r
mean(vote_yes)
```

```
## [1] 0.4533333
```

equals 

\begin{align}
\bar{X} &= \frac{1}{75}\sum_{i=1}^{75}x_i
\end{align}

It is rare though that we the have results from the full population. Due to varying rates of non-response to surveys, at best we might be able to assume we have a random sample for each demographic group within the population. This could mean that the size of the groups may not be in proportion to the the whole population. For example, suppose this is another survey on the same population, over demographics `A` and `B`:


```r
library(tidyverse)
poll_data <- tibble(
  group = c(rep("A", 25), rep("B", 10)),
  yes = c(rep(1, 19), rep(0, 6), rep(1, 3), rep(0, 7))
)
poll_data %>%
  group_by(group) %>%
  summarise(count = n())
```

```
## # A tibble: 2 x 2
##   group count
##   <chr> <int>
## 1     A    25
## 2     B    10
```

If we have some external Census data, we can easily tell that this survey is non-representative. 


```r
Census <- tibble(
  group = c("A", "B"),
  pop = c(25, 50)
)
```

We see this directly by comparing this mean


```r
mean(poll_data$yes)
```

```
## [1] 0.6285714
```

to the overall population mean shown above. 

How then do we estimate the overall population mean based on `poll_data`? We need to correct based on our Census data, knowing that each mean represents only a certain certaion of the population. This is poststratification. 

Let $X_a = x_{a1}, x_{a2}, \dots, x_{a25}$ and $X_b = x_{b1}, x_{b2}, \dots, x_{b50}$ be the data indicating support for each demographic. Then using $\bar{X}$, we have

\begin{align*}
\bar{X} &= \frac{1}{75}\sum_{i=1}^{75}x_i \\
&= \frac{1}{75}\left( \sum_{i=1}^{25}x_{ai} + \sum_{i=1}^{75}x_{bi} \right) \\
&= \frac{1}{75}\left(\frac{25}{25} \sum_{i=1}^{25}x_{ai} + \frac{50}{50}\sum_{i=1}^{75}x_{bi} \right) \\
&= \frac{1}{75} \left( 25 \bar{X}_a + 50 \bar{X}_b \right) \\
&= \frac{25}{75}\bar{X}_a + \frac{50}{75} \bar{X}_b
\end{align*}

Then assuming the samples from `poll_data` are random samples within each demographic group, their expected value should equal $\bar{X}_a$ and $\bar{X}_b$:


```r
group_support <- poll_data %>%
  group_by(group) %>%
  summarise(perc_support = mean(yes))
group_support
```

```
## # A tibble: 2 x 2
##   group perc_support
##   <chr>        <dbl>
## 1     A         0.76
## 2     B         0.30
```

```r
overall_support <- group_support %>%
  summarise(total_support = sum(perc_support * Census$pop/sum(Census$pop)))
overall_support
```

```
## # A tibble: 1 x 1
##   total_support
##           <dbl>
## 1     0.4533333
```


## Another example

Next, let's us a more complex example to demonstrate some alternative implementations of poststratification. Here, we will use the data from the book [Complex Surveys: A Guide to Analysis Using R](http://r-survey.r-forge.r-project.org/svybook/) by Thomas Lumley. 


```r
library(survey)
data(api)
```

The data set `api` is the California Academic Performance Index that surveys all 6194 California schools, which includes 4421 elementary schools, 755 high schools, and 1018 middle schools. This information will be our Census data.


```r
Census <- tibble(
  stype = c("E", "H", "M"),
  pop = c(4421, 755, 1018)
)
```

We will be working with a subset of that data


```r
d <- apistrat %>% as.tibble()
d %>% 
  group_by(stype) %>% 
  summarise(school_count = n())
```

```
## # A tibble: 3 x 2
##    stype school_count
##   <fctr>        <int>
## 1      E          100
## 2      H           50
## 3      M           50
```

which is not representative. For reference, on page 23 of his book, Lumley analyzes this data set using his package `survey`.


Since this school types are not in proportion to the total population, we need to do poststratification. Proceeding as before


```r
d.group.ave <- d %>% 
  group_by(stype) %>%
  summarise(ave_score = mean(api00))
d.group.ave
```

```
## # A tibble: 3 x 2
##    stype ave_score
##   <fctr>     <dbl>
## 1      E    674.43
## 2      H    625.82
## 3      M    636.60
```

```r
d.total.ave <- d.group.ave %>%
  summarise(ave_score = sum(ave_score * Census$pop/sum(Census$pop)))
d.total.ave
```

```
## # A tibble: 1 x 1
##   ave_score
##       <dbl>
## 1  662.2874
```

Conveniently, the data set `api` contains the scores for every school:


```r
apipop %>% as.tibble() %>%
  summarise(mean(api00))
```

```
## # A tibble: 1 x 1
##   `mean(api00)`
##           <dbl>
## 1      664.7126
```

which shows that our poststratification was a pretty good approximation. 

### The Total Statistic

So far we have been taking the weighted averages of subgroup averages to find the total population average. As you'll see in the MRP article, this is usually what we want. However, let's explore how poststratification works for the total statistics, and see how we can generalize the total statistic to other statistics that require a total.  

It doesn't make much sense to estimate the total test scores, but it does for enrollment. 

Let $X = x_1, x_2, \dots, x_{6194}$ be the students enrolled at each school. Then $T(X) = \sum_{i=1}^{6194}x_i$ is the total number of students enrolled. Then $T(X) = T(X_e) + T(X_h) + T(X_m)$ where $X_e, X_h, X_m$ are the enrollment data by school type. 

In our school survey, the total enrollment for each school type is


```r
d.group.enroll <- d %>% 
  group_by(stype) %>%
  summarise(enroll = sum(enroll))
d.group.enroll
```

```
## # A tibble: 3 x 2
##    stype enroll
##   <fctr>  <int>
## 1      E  41678
## 2      H  66035
## 3      M  41624
```

Again, it is clear we need a correction to find $T(X)$. If only 50 of the high schools surveyed have 66035 students enrolled the total enrollment for all 755 high schools, let alone all schools, will be much higher. 

In the previous section, we made the assumption that the expected mean of subgroup samples equals the mean of the subgroup. Now we need to make a similiar assumption: that the expected total enrollment in $n$ schools of a certain type is the same as another random sample of $n$ schools of the same type in the population. In other words, we need to assume that

\begin{align}
T(X_h) &= \sum_{i=1}^{755}x_{hi} \\
&= \sum_{i=1}^{50}x_{hi} + \sum_{i=51}^{100}x_{hi} + \cdots + \sum_{i=701}^{750}x_{hi} + \sum_{i=751}^{755}x_{hi}   \\
&\approx \frac{755}{50}\sum_{i = 1}^{50} \hat{x}_{hi}
\end{align}

where $\hat{X}_h = \hat{x}_{h1}, \hat{x}_{h2}, \dots, \hat{x}_{h50}$ is the enrollment data from the highschool survey subgroup. Previously we scaled down the averages in proportion to the whole, and now we are scaling up the totals relative to the subgroup size. 

Calculating,


```r
d.tot.enroll <- d.group.enroll %>%
  summarise(total_enroll = sum(enroll * Census$pop/c(100, 50, 50)))
d.tot.enroll
```

```
## # A tibble: 1 x 1
##   total_enroll
##          <dbl>
## 1      3687178
```

This type of poststratification gives us a pretty decent estimate.


```r
apipop %>% as.tibble() %>%
  summarise(total=sum(enroll, na.rm=TRUE))
```

```
## # A tibble: 1 x 1
##     total
##     <int>
## 1 3811472
```

### Generalizing

The weighted average of subgroup averages is actually a special case of this total-based poststratification method. Let's consider average test scores again, where $\hat{X}_e, \hat{X}_h, \hat{X}_m$ is the test score data for each school type in our survey, with $X_e, X_h, X_m$ being the population score data by school type. Then by definition, the average

\begin{align}
\bar{X} &= \frac{1}{6194}\left( T(X_e) + T(X_h) + T(X_m) \right) \\
&\approx \frac{1}{6194}\left(\frac{4421}{100} T(\bar{X}_e) + \frac{755}{50} T(\bar{X}_h) + \frac{1018}{50} T(\bar{X}_m)        \right) \\
&= \frac{4421}{6194} \frac{T(\bar{X}_e)}{100} + \frac{755}{6194} \frac{T(\bar{X}_h)}{50} + \frac{1018}{6194} \frac{T(\bar{X}_m)}{50}      \\
&\approx \frac{4421}{6194} \bar{X}_e + \frac{755}{6194} \bar{X}_h + \frac{1018}{6194} \bar{X}_m
\end{align}

which is the weighted average of subgroup averages found in the first section. 

In `R`,


```r
d.api.tot <- d %>%
  group_by(stype) %>%
  summarise(tot_api = sum(api00))
d.api.tot
```

```
## # A tibble: 3 x 2
##    stype tot_api
##   <fctr>   <int>
## 1      E   67443
## 2      H   31291
## 3      M   31830
```

```r
d.api.mean <- d.api.tot %>%
  summarise(api_mean = sum(tot_api * Census$pop/c(100, 50, 50))/6194)
d.api.mean
```

```
## # A tibble: 1 x 1
##   api_mean
##      <dbl>
## 1 662.2874
```

which is exactly the same mean we found before. 





