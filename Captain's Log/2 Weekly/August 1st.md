---
format: word_document
---

```{R, include = FALSE}
setwd("/home/aavila/forest_regrowth")

data <- readRDS('santoro_ESA_alldata.rds')

library(tidyverse)
```

Hi Brian and Fiona - here's me trying to figure out how to do this!

FYI Brian - on my folder in Janus there's all the code,

it's commented and there's a README file that may explain better

how things work.

  

I decided to go back to the drawing board a bit and sanity check every step of the way.

Starting with example data:

  

```{R}

set.seed(123)

  

x <- seq(0, 10, by = 0.1)

y <- 100 * (1 - exp(-0.2 * x)) + rnorm(length(x), sd = 5)

  

G <- function(pars) {

  pars['a'] * (1 - exp(-pars['b'] * x))

}

  

neg_log_likelihood <- function(pars) {

  -sum(dnorm(y, G(pars), sd = 1, log = TRUE))

}

  

optim(par = c(a = 1, b = 1), fn = neg_log_likelihood)

  

```

  

Behaves as expected, so far so good.

  

I noticed that if I try to fit the standard deviation value, predictions are terrible.

  

```{R}

  

neg_log_likelihood2 <- function(pars) {

  if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

    return(-Inf)

  }

  -sum(dnorm(y, G(pars), sd = pars['sd'], log = TRUE))

}

  

optim(par = c(a = 1, b = 1, sd = 1), fn = neg_log_likelihood2)

  

```

  

Fitting an asymptote and non-zero intercept:

  

```{R}

y <- 20 + 100 * (1 - exp(-0.2 * x)) + rnorm(length(x), sd = 5)

  

G <- function(pars) {

    pars['b_0'] + pars['a'] * (1 - exp(-pars['c']*x))

}

  

optim(par = c(a = 1, b_0 = 1, c = 1), fn = neg_log_likelihood)

  

```

  

Then we had six scenarios to explore:

  

Normalization (standard vs maxmin vs none), LU included (T or F), and st_dev fit or hard-coded.

  

Maxmin normalization, LU not included, st dev fit:

  

```{R}

setwd("/home/aavila/forest_regrowth")

data <- readRDS('santoro_ESA_alldata.rds')

  

data <- data[data$last_LU %in% c(15, 41, 48),]

data$last_LU <- factor(data$last_LU)

dummy_LU <- as.data.frame(model.matrix(~ data$last_LU - 1))

names(dummy_LU) <- c('pasture', 'other_annual', 'other_perennial')

data <- data[,-7]

  

minMax <- function(x) {

  (x - min(x)) / (max(x) - min(x))

}

data <- as.data.frame(lapply(data, minMax))

  

#data <- as.data.frame(scale(data))

  

data <- cbind(data, dummy_LU)

  

######################################################################

#################        passing into the model     ##################

######################################################################

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E

  (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.5, prec = 0.5, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

NLL = function(pars) {

if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

  return(-Inf)

}

Gpred = G(pars)

# Negative log-likelihood

-sum(dnorm(x = data$agbd - Gpred, mean = 0, sd = pars['sd'], log = TRUE), na.rm = TRUE)

}

  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par[1:length(o$par)])

  

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

The plot above looks the same when we fix sd at 1. - unsure as to why, could investigate later.

  

Maxmin normalization, LU included, st dev fit:

  

```{R, echo = FALSE}

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E + LU

  (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.5, prec = 0.5, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

  NLL = function(pars) {

  if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

    return(-Inf)

  }

  Gpred = G(pars)

  # Negative log-likelihood

  -sum(dnorm(x = data$agbd - Gpred, mean = 0, sd = pars['sd'], log = TRUE), na.rm = TRUE)

  }

  
  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par[1:length(o$par)])

  

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

  

I noticed the negative values were happening when the standard deviation was fixed for

the case of inclusion of LU - so that below is what happened that I showed before:

  

Maxmin normalization, LU included, st dev hard coded:

  

```{R, echo = FALSE}

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E + LU

  (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.5, prec = 0.5, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

  NLL = function(pars) {

  # if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

  #   return(-Inf)

  # }

  Gpred = G(pars)

  # Negative log-likelihood

  -sum(dnorm(x = data$agbd - Gpred, mean = 0, sd = 1, log = TRUE), na.rm = TRUE)

  }

  
  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par[1:length(o$par)])

  

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

  

Untransformed data, LU not included, st dev fit:

  All predicted values are the same.

  

The initial parameter for temperature and precipitation had to be a very tiny value,

as the values themselves are large.

  

For some reason, optim deviated from that and settled on larger values for the parameters,

which blew up the exponent, and a bunch of -Inf resulted.

  

Untransformed data, LU not included, st dev fixed:

  

```{R, echo = FALSE}

setwd("/home/aavila/forest_regrowth")

data <- readRDS('santoro_ESA_alldata.rds')

  

data <- data[data$last_LU %in% c(15, 41, 48),]

data$last_LU <- factor(data$last_LU)

dummy_LU <- as.data.frame(model.matrix(~ data$last_LU - 1))

names(dummy_LU) <- c('pasture', 'other_annual', 'other_perennial')

data <- data[,-7]

  

minMax <- function(x) {

  (x - min(x)) / (max(x) - min(x))

}

#data <- as.data.frame(lapply(data, minMax))

  

#data <- as.data.frame(scale(data))

  

data <- cbind(data, dummy_LU)

  

######################################################################

#################        passing into the model     ##################

######################################################################

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E

  pars['B_0'] + pars['A'] * (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.0005, prec = 0.000005, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

Gpred <- G(pars)

  

NLL = function(pars) {

-sum(dnorm(x = data$agbd - Gpred, mean = 0, sd =  1, log = TRUE), na.rm = TRUE)

}

  
  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par)

  

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

  
  

Untransformed data, LU included, st dev fixed:

  

```{R, echo = TRUE}

setwd("/home/aavila/forest_regrowth")

data <- readRDS('santoro_ESA_alldata.rds')

  

data <- data[data$last_LU %in% c(15, 41, 48),]

data$last_LU <- factor(data$last_LU)

dummy_LU <- as.data.frame(model.matrix(~ data$last_LU - 1))

names(dummy_LU) <- c('pasture', 'other_annual', 'other_perennial')

data <- data[,-7]

  

minMax <- function(x) {

  (x - min(x)) / (max(x) - min(x))

}

#data <- as.data.frame(lapply(data, minMax))

  

#data <- as.data.frame(scale(data))

  

data <- cbind(data, dummy_LU)

  

######################################################################

#################        passing into the model     ##################

######################################################################

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E + LU

  pars['B_0'] + pars['A'] * (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.0005, prec = 0.000005, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

Gpred <- G(pars)

head(Gpred)

  

NLL = function(pars) {

# if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

#   return(-Inf)

# } pars['sd']

# Gpred = G(pars)

# Negative log-likelihood

-sum(dnorm(x = data$agbd - Gpred, mean = 0, sd =  1, log = TRUE), na.rm = TRUE)

}

  
  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par)

  

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

  

Standard scaling, LU not included, sd hard coded:

  

```{R, echo = TRUE}

setwd("/home/aavila/forest_regrowth")

data <- readRDS('santoro_ESA_alldata.rds')

  

data <- data[data$last_LU %in% c(15, 41, 48),]

data$last_LU <- factor(data$last_LU)

dummy_LU <- as.data.frame(model.matrix(~ data$last_LU - 1))

names(dummy_LU) <- c('pasture', 'other_annual', 'other_perennial')

data <- data[,-7]

  

minMax <- function(x) {

  (x - min(x)) / (max(x) - min(x))

}

#data <- as.data.frame(lapply(data, minMax))

  

data <- as.data.frame(scale(data))

  

data <- cbind(data, dummy_LU)

  

######################################################################

#################        passing into the model     ##################

######################################################################

  

G <- function(pars) {

  E = pars['temp'] * data$temp + pars['prec'] * data$prec

  LU = pars['total_fires'] * data$total_fires + pars['ts_fire'] * data$ts_fire + pars['pasture'] * data$pasture +

      pars['other_perennial'] * data$other_perennial + pars['other_annual'] * data$other_annual

  k = E

  pars['B_0'] + pars['A'] * (1 - exp(-k))

}

  

pars = c(B_0 = 10, A = 100, temp = 0.0005, prec = 0.000005, total_fires = 0.05, ts_fire = 0.05, pasture = 0.05, other_perennial = 0.05, other_annual = 0.05,  sd = 0.05)

  

Gpred <- G(pars)

head(Gpred)

  

NLL = function(pars) {

# if(pars['sd'] < 0){ #avoiding NAs by keeping the st dev positive

#   return(-Inf)

# } pars['sd']

# Gpred = G(pars)

# Negative log-likelihood

-sum(dnorm(x = data$agbd - Gpred, mean = 0, sd =  1, log = TRUE), na.rm = TRUE)

}

  
  

o = optim(par = pars, fn = NLL, hessian = FALSE)

o

  

pred = G(o$par)

outcome <- data.frame(data$agbd, pred)

outcome <- round(outcome, 3)

  

median_values <- outcome %>%

  group_by(pred) %>%

  summarize(median_agbd = median(data.agbd, na.rm = TRUE))

  

plot(median_values$pred, median_values$median_agbd, abline(0,1))

  

```

  

Okay, so standard scaling introduces negative values, which really doesn't work well

with the equation, which is understandable.

  
  

# What's going on with Heinrich's model?

  

So, turns out the models are available as .Rdata files.

  

```{R}

setwd("/home/aavila/forest_regrowth/heinrich_poorter/Fig1_data_input/Fig1_data_input/")

  

load("regrowth_model_rad_187.RData")

model_CR_rad187

class(model_CR_rad187)

  

load("regrowth_model_precip_1920.RData")

model_CR_precip1920

```

  
  

# Incorporating mean biomass of surrounding mature forests:

  

Initially we would just incorporate the biomass of the nearest

mature forest patch, but actually having a mean of the surrounding

area would actually be a better idea.

Trying to execute this, the terra function that would work is

focal():

  

mature_sum <- focal(mature_mask, range, sum, na.rm = TRUE) # run focal on these areas

  

this works well for general area (looking at the total cover of mature forests

around, when the mask contains 1 and 0 values), but it's too slow when the

values are biomasses (to get the mean or even total biomass around).

  

mature_biomass is a raster with biomasses of the mature forests around.

I thought about making a buffer around secondary areas and masking the

map of mature forests, but that's being too slow with terra. I'm currently working

on trying to do it with matrices and/or parallelize it.

  
  

# Climatic data

  

Heinrich 2021 and Poorter 2017, the most similar papers to our model, don't use temperature,

but they don't use actual evapotranspiration either:

  

- Poorter used CWD from the Chave et al 2014 paper, and rainfall from WorldClim.

- Heinrich used:

    - radiation from terraclimate

    - Precipitation from this paper (https://www.nature.com/articles/sdata201566)

    and used an R script to derive Maximum CWD from that.

    - Cation concentration from a different paper

  

Both CWD and Evapotranspiration can be derived from precipitation data.

  

I don't know why she did not use CWD and precipitation from TerraClimate, as she

got radiation data from there?

  
  
  

# Ideas on what to do for the coming week:

  

- Finish exploration of Heinrich 2021 data - try and reproduce the results with nls function?

- Take a look at TerraClimate AET and CWD - how is it derived? Could this be an option?

- Fix the mature forest max biomass script