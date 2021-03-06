Analysis of T<sub>RM</sub> population dynamics
================

<!--<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  TeX: { equationNumbers: { autoNumber: "AMS" } }
});
</script>-->
This approach illustrates how a simple mathematical model can be applied to experimental data of T<sub>RM</sub> in nonlymphoid tissues. The aim is to estimate the clonal half-life of each population and project dynamics beyond experimental timeframes. For full details of the model, see Morris et al (2019) Journal of Immunology ([doi:10.4049/jimmunol.1900767](https://doi.org/10.4049/jimmunol.1900767)). Details of the data collection methods can be found in:

-   Pizzolla et al (2017) Science Immunology, [doi:10.1126/sciimmunol.aam6970](https://immunology.sciencemag.org/content/2/12/eaam6970)
-   Slutter et al (2017) Science Immunology, [doi:10.1126/sciimmunol.aag2031](https://immunology.sciencemag.org/content/2/7/eaag2031)

### Modeling approach

A simple model of homeostasis assumes new T<sub>RM</sub> cells (*T*) are recruited from a precursor subset at rate *r*, proliferate at rate *p*, and decay at rate *d*. These dynamics be expressed mathematically as

<!--$$\begin{aligned} \frac{dT}{dt} & = r + p T - d T \\ & = r - (d - p)T, \end{aligned}$$-->
<!--$$\frac{dT}{dt} = r - (d - p)T,$$-->
<sup>dT</sup>⁄<sub>dt</sub> = r - (d - p)T,

where *d* − *p* is the net loss rate from the population. If there is no recruitment from precursor populations, then *r* = 0 and the above equation can be solved to give

*T* = *T*<sup>0</sup>exp<sup>−(*d* − *p*)*t*</sup>,

where *T*<sup>0</sup> is the initial number of T<sub>RM</sub>. Thus, assuming cells are lost at a faster rate than they divide (i.e. *d* &gt; *p*), the clonal population will decline exponentially at rate (*d* − *p*), with an average half-life of log(2)/(*d* − *p*).

T<sub>RM</sub> loss can be modeled by log-transforming this second equation to obtain a linear regression model with intercept log(*T*<sup>0</sup>) and slope −(*d* − *p*), i.e.

log(*T*)=log(*T*<sup>0</sup>)−(*d* − *p*)*t*.

Using least squares optimization, we fit this equation to the log-transformed measurements from each dataset and estimate the corresponding net loss rate and clonal half-life. Note that in a more thorough analysis, this model would be compared with alternative models to assess the most parsimonious description of the data.

### Analysis

To implement the analysis in R, we first load the required packages:

``` r
require(tidyverse)
require(cowplot)
```

And define settings for the final plot:

``` r
basetext <- 10
mytheme <- ggthemes::theme_tufte(base_family = "Helvetica") + 
    theme(axis.text = element_text(size = basetext), 
          axis.title = element_text(size = basetext + 0.5),
          legend.text = element_text(size = basetext),
          legend.title = element_text(size = basetext + 0.5),
          strip.text.x = element_text(size = basetext + 0.5), 
          panel.border = element_rect(colour = "black", fill = NA))

ybreaks <- c(1e0, 1e2, 1e4, 1e6)
ylabels <- expression(1, 10^2, 10^4, 10^6)
ytitle <- expression(CD69^{"+"}~CD103^{"+"}~T[RM])

textsize <- 10
```

We then define functions that will be used in the analysis. Note that for the response variable we use the approximation log(T<sub>RM</sub> + 1) to avoid NAs when T<sub>RM</sub> = 0. This does not substantially impact our results, and is preferable to omitting important information contained in those undetectable measurements.

``` r
# 1. Estimate parameters and 95% CIs by fitting linear model

getparams <- function(data){
    lmfit <- lm(log(Trm + 1) ~ day, data = data)
    
    netloss <- - coef(lmfit)[2]
    
    return( data.frame(netloss = netloss, halflife = log(2)/netloss) )
}

# 2 Test for normal residuals from fitted linear model
testresids <- function(data){
    lmfit <- lm(log(Trm + 1) ~ day, data = data)
    
    pval <- shapiro.test(lmfit$residuals)$p.value
    
    return(data.frame(pval = pval) )
}

# 3. Get predictions from fitted linear model
getpredictions <- function(data, xnew){
    lmfit <- lm(log(Trm + 1) ~ day, data = data)
    
    predictions <- data.frame(day = xnew, Trm = predict(lmfit, xnew, interval = "confidence")) %>% 
        mutate(Trm = exp(Trm.fit) - 1, upper = exp(Trm.upr) - 1, lower = exp(Trm.lwr) - 1)
    
    return(predictions)
}
```

After this, we load the data:

``` r
load("data_pizzolla.RData")
load("data_slutter.RData")
```

And carry out analysis for each data set:

``` r
## Pizzolla et al analysis ----------------------------------

# 1. Model fitting
params_pizzolla <- data_pizzolla %>% group_by(tissue, study) %>% do(getparams(data = .) )

resids_pizzolla <- data_pizzolla %>% group_by(tissue, study) %>% do(testresids(data = .) )

# 2. Model predictions
xnew <- data.frame(day = seq(0, 250, by = 0.5))

predict_pizzolla <- data_pizzolla %>% group_by(tissue, study) %>% 
    do(getpredictions(data = ., xnew = xnew) ) %>% ungroup()
```

``` r
## Slutter et al analysis ----------------------------------

# 1. Model fitting
params_slutter <- data_slutter %>% group_by(tissue, study) %>% do(getparams(data = .) )

resids_slutter <- data_slutter %>% group_by(tissue, study) %>% do(testresids(data = .) )

# 2. Model predictions
predict_slutter <- data_slutter %>% group_by(tissue, study) %>% 
    do(getpredictions(data = ., xnew = xnew) ) %>% ungroup()
```

Then we plot the results:

``` r
plot_pizzolla <- ggplot(data = data_pizzolla, aes(x = day, y = Trm + 1)) + 
    facet_wrap(~ tissue) +
    geom_ribbon(data = predict_pizzolla, aes(x = day, ymin = lower + 1, ymax = upper + 1), 
                fill = "grey") +
    geom_line(data = predict_pizzolla) + geom_point(alpha = 0.5) +
    geom_text(data = params_pizzolla, 
              aes(x = 65, y = 2.2e-1, 
                  label = paste("Population half-life =", signif(halflife, 2), "days")), 
              size = textsize/.pt) + 
    mytheme + scale_x_continuous("Days post infection") + 
    scale_y_continuous(ytitle, breaks = ybreaks, labels = ylabels, trans = "log")

plot_slutter <- ggplot(data = data_slutter, aes(x = day, y = Trm + 1)) + 
    facet_wrap(~ tissue) + 
    geom_ribbon(data = predict_slutter, aes(x = day, ymin = lower + 1, ymax = upper + 1), 
                fill = "grey") +
    geom_line(data = predict_slutter) + geom_point(alpha = 0.5) + 
    geom_text(data = params_slutter, 
              aes(x = 100, y = 1.2e-1, 
                  label = paste("Population half-life =", signif(halflife, 2), "days")), 
              size = textsize/.pt) + 
    mytheme + scale_x_continuous("Days post infection") +
    scale_y_continuous(ytitle, breaks = ybreaks, labels = ylabels, 
                       limits = c(3e-2, 1e6), trans = "log")

print(
    plot_all <- plot_grid(plot_pizzolla, plot_slutter, 
                          nrow = 2, labels = c("A", "B"), label_size = 12)
)
```

![](README_files/figure-markdown_github/plotting-1.png)

Finally, to estimate the time at which OT-I (Panel A) and NP366 (Panel B) T<sub>RM</sub> in the lung become undetectable, we simply find the first time at which their predicted population sizes fall below one:

``` r
predict_pizzolla %>% filter(tissue == "Lung", Trm < 1) %>% 
    slice(1) %>% mutate(day = signif(day, 2)) %>% select(day)
```

    ## # A tibble: 1 x 1
    ##     day
    ##   <dbl>
    ## 1   220

``` r
predict_slutter %>% filter(tissue == "Lung (NP366)", Trm < 1) %>% 
    slice(1)  %>% mutate(day = signif(day, 2)) %>% select(day)
```

    ## # A tibble: 1 x 1
    ##     day
    ##   <dbl>
    ## 1   210
