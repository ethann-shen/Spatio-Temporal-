---
title: "Lecture 13 - Gaussian Process Models Part 2"
author: "Ethan Shen"
date: "5/25/2020"
output: 
  html_document:
    keep_md: TRUE
---




```r
vals = expand.grid(
  d = seq(0, 2, length.out = 100),
  l = seq(1, 7, length.out = 10)
) %>%
  as.data.frame() %>%
  tbl_df()
```

## Covariance vs Semivariogram - Exponential and Squared Exponential Covariance Functions


```r
exp = rbind(
  mutate(vals, func="exp cov", y = exp_cov(d, l=l)),
  mutate(vals, func="exp semivar", y = exp_sv(d, l=l))
) 

sq_exp = rbind(
  mutate(vals, func="sq exp cov", y = sq_exp_cov(d, l=l)),
  mutate(vals, func="sq exp semivar", y = sq_exp_sv(d, l=l))
) 

rbind(exp, sq_exp) %>%
  mutate(l = as.factor(round(l,1))) %>%
  ggplot(aes(x=d, y=y, color=l)) +
  geom_line() +
  facet_wrap(~func, ncol=2) + 
  labs(title=expression(paste(sigma[w]^2, " = 0")))
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

We are interested in asymptotic properties of these functions. 
  
**When examining a semivariogram, we should limit ourselves to the "first half". This where we have the most data points**
  
# Example from Last Time 


```r
load(file="lec12_ex.Rdata")
base
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

### Empirical Semivariograms


```r
library(geoR)
coords = d$t %>% dist() %>% as.matrix()
dist = fields::rdist(coords)
par(mfrow=c(2,2))
variog(coords=d$t %>% dist() %>% as.matrix(), data = d$y, messages = FALSE,
       uvec = seq(0, max(dist), length.out=100)) %>% plot(main = "100")
variog(coords=coords, data = d$y, messages = FALSE,
       uvec = seq(0, max(dist), length.out=75)) %>% plot(main = "75")
variog(coords=coords, data = d$y, messages = FALSE,
       uvec = seq(0, max(dist), length.out=50)) %>% plot(main = "50")
variog(coords=d$t %>% dist() %>% as.matrix(), data = d$y, messages = FALSE,
       uvec = seq(0, max(dist), length.out=30)) %>% plot(main = "30")
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-4-1.png)<!-- -->

```r
d_emp = rbind(
  d %>% emp_semivariogram(y, t, bin=TRUE, binwidth=0.05)  %>% mutate(binwidth="binwidth=0.05"),
  d %>% emp_semivariogram(y, t, bin=TRUE, binwidth=0.075) %>% mutate(binwidth="binwidth=0.075"),
  d %>% emp_semivariogram(y, t, bin=TRUE, binwidth=0.1)   %>% mutate(binwidth="binwidth=0.1"),
  d %>% emp_semivariogram(y, t, bin=TRUE, binwidth=0.15)  %>% mutate(binwidth="binwidth=0.15")
)

d_emp %>%
  ggplot(aes(x=h, y=gamma, size=n)) +
  geom_point() +
  facet_wrap(~binwidth, nrow=2)
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-4-2.png)<!-- -->

## Theoretical vs Empirical Semivariogram 

Parameter values from last time: 


```r
params_lec12 = readRDS("lec_12_params")
l = params_lec12[1] ; sigma2 = params_lec12[3] ; sigma2_w = params_lec12[2]

d_fit = data_frame(h=seq(0, 1 ,length.out = 100)) %>%
  mutate(gamma = sigma2 + sigma2_w - (sigma2 * exp(-(l*h)^2)))

d_emp %>%
  filter(binwidth %in% c("binwidth=0.05", "binwidth=0.1")) %>%
  ggplot(aes(x=h, y=gamma)) +
  geom_point() +
  geom_line(data=d_fit, color='red') +
  geom_point(x=0,y=sigma2_w, color='red') +
  facet_wrap(~binwidth, ncol=2)
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

These semivariograms look pretty good. 

# Example: FRN Data


```r
load("frn_example.Rdata")

pm25 = pm25 %>%
  mutate(Date = lubridate::mdy(Date)) %>%
  mutate(day  = (Date - lubridate::mdy("1/1/2007") + 1) %>% as.integer()) %>% 
  select(-POC) %>%
  setNames(., tolower(names(.)))

ggplot(pm25, aes(x=date, y=pm25)) +
  geom_line() +
  geom_point()
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

We cannot fit an ARIMA because there is lots of missing data. The data should be collected every 3 days, but that's not the case. We can to use a Gaussian Process model. 

## Empirical Variogram


```r
rbind(
  pm25 %>% emp_semivariogram(pm25, day, bin=TRUE, binwidth=3) %>% mutate(bw = "binwidth=3"),
  pm25 %>% emp_semivariogram(pm25, day, bin=TRUE, binwidth=6) %>% mutate(bw = "binwidth=6")
) %>%
  ggplot(aes(x=h, y=gamma, size=n)) +
    geom_point(alpha=0.5) +
    facet_wrap(~bw, ncol=2)
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

This doesn't have a great structure. It doesn't really plateau. We want get a better picture at the trend. Usually, in ARIMA, we would difference but we can't here, because we don't have a constant time interval. 

## Quadratic Model


```r
ggplot(pm25, aes(x=day, y=pm25)) +
  geom_line() +
  geom_point() +
  geom_smooth(method="lm", formula=y~x+I(x^2), se=FALSE, color="red", fullrange=TRUE)
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

```r
(l=lm(pm25~day+I(day^2), data=pm25))
```

```
## 
## Call:
## lm(formula = pm25 ~ day + I(day^2), data = pm25)
## 
## Coefficients:
## (Intercept)          day     I(day^2)  
##  12.9644351   -0.0724639    0.0001751
```

```r
l
```

```
## 
## Call:
## lm(formula = pm25 ~ day + I(day^2), data = pm25)
## 
## Coefficients:
## (Intercept)          day     I(day^2)  
##  12.9644351   -0.0724639    0.0001751
```

We can look at the empirical semivariogram of the residuals to see if they are performing better. 


```r
pm25 = pm25 %>%
  modelr::add_residuals(l)

rbind(
  pm25 %>% emp_semivariogram(resid, day, bin=TRUE, binwidth=3) %>% mutate(bw = "binwidth=3"),
  pm25 %>% emp_semivariogram(resid, day, bin=TRUE, binwidth=6) %>% mutate(bw = "binwidth=6")
) %>%
  ggplot(aes(x=h, y=gamma, size=n)) +
    geom_point(alpha=0.5) +
    facet_wrap(~bw, ncol=2)
```

![](Lec13-GP-Models-Pt-2_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

Generally, we want to limit ourselves to the first half of the semivariogram, because that's where we have the most data. This looks better. 

In this semivariogram, the nugget is pretty large (~10), but the sill is ~12ish, meaning sigma2 is ~2. This means that most of the noise is unexplained white noise (noise of process), and only a little explained by the covariance function.
