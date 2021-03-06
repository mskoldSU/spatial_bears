Simple spatial effort and density
================

A simple illustration of how to obtain measures of effort, density and
abundance with minimal coding based on Swedish brown bear monitoring
data.

Load data (`n` is number of captures) and a grid to predict on.

``` r
library(tidyverse)
data <- read_csv("bear_captures.csv") %>% 
  filter(year == 2020)
data
```

    ## # A tibble: 1,154 x 5
    ##    sex    year     n     lon      lat
    ##    <chr> <dbl> <dbl>   <dbl>    <dbl>
    ##  1 Hona   2020     1 482193  7139709 
    ##  2 Hona   2020     2 449493  6870228.
    ##  3 Hona   2020     3 662407. 7031803 
    ##  4 Hona   2020     3 569840. 7046490 
    ##  5 Hona   2020     1 588215  7094753 
    ##  6 Hona   2020     2 582999  7068861 
    ##  7 Hona   2020     8 678593. 7042886.
    ##  8 Hona   2020     4 550818. 7083243.
    ##  9 Hona   2020     2 587833  7084893 
    ## 10 Hona   2020     3 388036  6932592.
    ## # ... with 1,144 more rows

``` r
predict_grid <- read_csv("grid_ZY.csv") # Just a spatial grid for visualisations
```

Fit a spatial GAM (thin-plate spline) with number of captures as
response, including some ad-hoc fixes to account for the lack of a
zero-truncated Poisson family in `mgcv`.

``` r
initial_fit <- mgcv::gam((n - 1) ~ s(lon, lat, k = 100), 
                         family = "poisson", 
                         data = data)
mu_tilde <- predict(initial_fit, 
                    newdata = predict_grid,
                    type = "response") + 1
fit_mu <- Vectorize(function(xbar){
  # Fits a zero-truncated Poisson based on observed mean
  uniroot(f = function(mu) mu / (1 - exp(-mu)) - max(xbar, 1.05), 
          interval = c(0.01, 100))[["root"]]
})
mu <- fit_mu(mu_tilde) # Mean number of captures
q <- 1 - exp(-mu) # Probability of capture
```

Fit a spatial intensity using a kernel density estimate.

``` r
H <- ks::Hpi.diag(cbind(data$lon, data$lat))
lambda_q <- ks::kde(cbind(data$lon, data$lat), H = H, 
                    eval.points = predict_grid)[["estimate"]] * nrow(data)
lambda <- lambda_q / q # Animal density
```

Illustrate search effort (as mean number of captures).

``` r
predict_grid %>% 
  ggplot(aes(x = lon, y = lat)) + 
  geom_contour_filled(aes(z = mu)) + 
  geom_point(data = data, color = "white", size = .01) +
  theme_void()
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

Illustrate probability of capture.

``` r
predict_grid %>% 
  ggplot(aes(x = lon, y = lat)) + 
  geom_contour_filled(aes(z = q)) +
  geom_point(data = data, color = "white", size = .01) + 
  theme_void()
```

![](README_files/figure-gfm/unnamed-chunk-5-1.png)<!-- -->

Animal density (individuals per km2).

``` r
predict_grid %>% 
  ggplot(aes(x = lon, y = lat)) + 
  geom_contour_filled(aes(z = lambda * 10^6)) + theme_void() +
  geom_point(data = data, color = "white", size = .01) +
  theme_void()
```

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

Estimating abundance.

``` r
mu_tilde_obs <- predict(initial_fit, type = "response") + 1
mu_obs <- fit_mu(mu_tilde_obs)
q_obs <- 1 - exp(-mu_obs)
sum(1 / q_obs) %>% round()
```

    ## [1] 1412
