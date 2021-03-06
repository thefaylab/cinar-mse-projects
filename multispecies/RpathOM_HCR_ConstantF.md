RpathOM
================
Sean Lucey
1/12/2022

# Use Rpath as the Operating model

To save time rather than develop a complicated multispecies model we can
use the existing Georges Bank Rpath model. Rpath is a mass balance
representation of the ecosystem. Extentions to extend the Rpath
framework as an operating model are outlined in Lucey et al. 2021.
Basically a step function is called that will pause the simulation,
allow for the evaluation of external assessment models that can then
inform a harvest control rule, and finally modify the parameters of the
model before contnuing with the simulation.

Mass balance is governed by two master equations; one describing
production:

\[ P_i = Y_i + B_iM2_i + E_i + BA_i + P_i(1 - EE_i) \]

where production, \(P_i\), is the sum of a species’ fishery yield,
\(Y_i\); predation mortality, \(M2_i\); emigration, \(E_i\); biomass
accumulation, \(BA_i\); and other mortality, expressed as
\(P_i(1-EE_i)\). Where \(EE_i\) is the ecotrophic efficiency or
percentage of mortality explained within the model. In this equation,
\(M2_i\) is expressed as a rate and therefore multiplied by the species
biomass, \(B_i\).

and the other consumption:

\[Q_i = P_i + R_i + U_i\]

where \(Q_i\) is the total consumption, \(R_i\) is respiration, and
\(U_i\) is unassimilated food. Energy used for respiration is lost from
the system. Unassimilated food is the portion of consumption that is
excreted and remains in the system through a detrital group.

``` r
#Load GB parameters
load(here('multispecies', 'data-raw', 'GB_balanced_params.RData'))
#If you want to see how the sausage was made check out the NOAA-EDAB/GBRpath repository on GitHub

#Generate balanced model
GB <- Rpath::rpath(GB.params, 'Georges Bank')
```

The Georges Bank Rpath model consists of 69 groups.  
Of these, 57 are living groups comprised of individual fish and
invertebrate species, aggregate fish and invertebrate groups, marine
mammals, birds, primary and secondary producers, and bacteria. There are
two detrital groups representing discards and general detritus. Finally,
there are 10 fleets representing the various fishing gears used on
Georges Bank. The resultant food web is highly interconnect, an expected
result due to the generalist nature of many of the species.

``` r
set.seed(123)
Group1 <- c(60, 58, 59)
Group2 <- c(50, 51, 55, 49, 53, 47, 52, 56, 57)
Group3 <- c(36, 48, 30, 11, 25, 34, 10, 32, 54, 33, 46, 22)
Group4 <- c(7, 43, 27, 38, 9, 8, 40, 44, 18, 14, 31, 41, 45, 35)
Group5 <- c(23, 1, 3, 37, 20, 13, 17, 39, 21, 19, 5, 42, 28, 26)
Group6 <- c(2, 4, 6, 12, 15, 16, 24, 29)
my.order <- c(Group1, Group2, Group3, Group4, Group5, Group6)
webplot(GB, labels = T, box.order = my.order)
```

![ Food web of the Georges Bank Rpath
model.](RpathOM_HCR_ConstantF_files/figure-gfm/Webplot-1.png)

# Generating initial data

Due to the nature of this model, it is not conditioned to data but
rather strictly a simulation. However, we simulated a range of fishing
pressures in order to generate initial data points that could be used by
assessment models.

``` r
#Add variable fishing in a dynamic run
GB.scene <- rsim.scenario(GB, GB.params, years = 1983:2022)
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'OtterTrawlLg',
                            sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                            rep(5, 5), rep(10, 5),
                                                            rep(2.5, 5)))
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'Gillnet',
                           sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                           rep(5, 5), rep(10, 5),
                                                           rep(2.5, 5)))
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'Midwater',
                           sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                           rep(5, 5), rep(10, 5),
                                                           rep(2.5, 5)))

GB.run <- rsim.run(GB.scene, years = 1983:2022)
```

Let’s look at the relative biomass from these initial conditions

``` r
#Plot results
GB.bio <- as.data.table(GB.run$out_Biomass)
GB.bio[, Month := 1:nrow(GB.bio)]
GB.bio <- data.table::melt(GB.bio, id.vars = 'Month', variable.name = 'Group', 
                           value.name = 'Biomass')
start <- data.table::as.data.table(GB.run$start_state$Biomass)
start[, Group := names(GB.run$start_state$Biomass)]

GB.bio <- merge(GB.bio, start, by = 'Group')
GB.bio[, Rel.biomass := Biomass / V1]

ggplot(GB.bio[Group %in% c('Cod', 'Haddock', 'AtlHerring')],
       aes(x = Month, y = Rel.biomass, col = Group)) +
  geom_line()
```

![](RpathOM_HCR_ConstantF_files/figure-gfm/Rel%20Biomass-1.png)<!-- -->

Need a function to pull observed data from the system:

``` r
GenData <- function(Rsim.output, group, sim.year, Sigma = 0.3, bias = 1, freq = 1){
  # This routine generates data (unbiased generally) for the biological groups 
  #and returns the appropriate object
  
  node     <- extract.node(Rsim.output, group)
  TrueBio  <- node$AnnualBiomass[which(names(node$AnnualBiomass) == sim.year)]
  Observed <- c()
  for(Iyear in seq_along(sim.year)){
    if (Iyear %% freq == 0){
      Observed[Iyear] <- TrueBio[Iyear] * exp(rnorm(1, 0, Sigma) - Sigma^2/2)
    } else Observed[Iyear] <- -1
  }
  Catch <- as.numeric(node$AnnualTotalCatch[which(names(node$AnnualBiomass) == sim.year)])
  
  out <- list(ObsBio = Observed, TotCatch = Catch, Fmort = Catch / Observed)
  
  return(out)
} 
```

Now generate the initial data set and fit to a Schaefer production model

``` r
setkey(123)
cod <- GenData(GB.run, 'Cod', 1983:2022, Sigma = 0.1)  
haddock <- GenData(GB.run, 'Haddock', 1983:2022, Sigma = 0.1)
atlherring <- GenData(GB.run, 'AtlHerring', 1983:2022, Sigma = 0.1)

#Merge into 1 file
GB.data <- data.table(Year = 1983:2022,
                      Species = c(rep('Cod', 40), rep('Haddock', 40), 
                                  rep('AtlHerring', 40)),
                      ObsBio = c(cod$ObsBio, haddock$ObsBio, atlherring$ObsBio),
                      TotCatch = c(cod$TotCatch, haddock$TotCatch, atlherring$TotCatch),
                      Fmort = c(cod$Fmort, haddock$Fmort, atlherring$Fmort))

#plot
GB.data %>%
  ggplot()+
  geom_point(aes(x=Year, y=ObsBio)) +
  geom_line(aes(x=Year, y=TotCatch)) +
  theme_bw() +
  facet_wrap(~Species, scales = "free")
```

![](RpathOM_HCR_ConstantF_files/figure-gfm/Initial%20data-1.png)<!-- -->

### Schaefer model (single species, directly from [my-first-mse](https://github.com/gavinfay/cinar-mse/blob/main/materials/exercises/day-01/my-first-mse.Rmd))

First, the logistic production function:

``` r
schaefer <- function(B,C,K,r) {
  #function schaefer takes the current biomass, a catch, 
  #and the model parameters to compute next year's biomass
  res <- B + B * r * (1 - B/K) - C
  return(max(0.001,res))  # we add a constraint to prevent negative biomass
}
```

Now a function to do the biomass projection:

``` r
dynamics <- function(pars,C,yrs) {
  # dynamics takes the model parameters, the time series of catch, 
  # & the yrs to do the projection over
  
  # first extract the parameters from the pars vector (we estimate K in log-space)
  K <- exp(pars[1])
  r <- exp(pars[2])
  
  # find the total number of years
  nyr <- length(C) + 1
  
  # if the vector of years was not supplied we create 
  # a default to stop the program crashing
  if (missing(yrs)) yrs <- 1:nyr
  
  #set up the biomass vector
  B <- numeric(nyr)
  
  #intialize biomass at carrying capacity
  B[1] <- K
  # project the model forward using the schaefer model
  for (y in 2:nyr) {
    B[y] <- schaefer(B[y-1],C[y-1],K,r)
  }
  
  #return the time series of biomass
  return(B[yrs])
  
  #end function dynamics
}  
```

We are going to condition the operating model by estimating the
parameters based on the historical biomass index data.

To do this we make a function that shows how well the current parameters
fit the data, we assume that the observation errors around the true
biomass are log-normally distributed.

``` r
# function to calculate the negative log-likelihood
nll <- function(pars,C,U) {  #this function takes the parameters, the catches, and the index data
  sigma <- exp(pars[3])  # additional parameter, the standard deviation of the observation error
  B <- dynamics(pars,C)  #run the biomass dynamics for this set of parameters
  Uhat <- B   #calculate the predicted biomass index - here we assume an unbiased absolute biomass estimate
  output <- -sum(dnorm(log(U),log(Uhat),sigma,log=TRUE),na.rm=TRUE)   #calculate the negative log-likelihood
  return(output)
  #end function nll
}
```

Function to perform the assessment and estimate the operating model
parameters  
(i.e. to fit the logistic model to abundance data)

``` r
assess <- function(catch,index,calc.vcov=FALSE,pars.init) {
  # assess takes catch and index data, initial values for the parameters,
  # and a flag saying whether to compute uncertainty estimates for the model parameters
  
  #fit model
  # optim runs the function nll() repeatedly with differnt values for the parameters,
  # to find the values that give the best fit to the index data
  res <- optim(pars.init,nll,C=catch,U=index,hessian=TRUE)
  
  # store the output from the model fit
  output <- list()
  output$pars <- res$par
  output$biomass <- dynamics(res$par,catch)
  output$convergence <- res$convergence
  output$nll <- res$value
  if (calc.vcov)
    output$vcov <- solve(res$hessian)
  
  return(output)
  #end function assess
}
```

### Single species models?

Try fitting to each species separately first, adjusting Gavin’s code
(test with atlantis output till we get rpath):

Make separate data objects for each species

``` r
harvest.GBcod <- GB.data %>%
  filter(Species=="Cod") %>%
  select(TotCatch) %>%
  unlist() %>%
  unname()

index.GBcod <- GB.data %>%
  filter(Species=="Cod") %>%
  select(ObsBio) %>%
  unlist() %>%
  unname()

harvest.GBherring <- GB.data %>%
  filter(Species=="AtlHerring") %>%
  select(TotCatch) %>%
  unlist() %>%
  unname()

index.GBherring <- GB.data %>%
  filter(Species=="AtlHerring") %>%
  select(ObsBio) %>%
  unlist() %>%
  unname()

harvest.GBhaddock <- GB.data %>%
  filter(Species=="Haddock") %>%
  select(TotCatch) %>%
  unlist() %>%
  unname()

index.GBhaddock <- GB.data %>%
  filter(Species=="Haddock") %>%
  select(ObsBio) %>%
  unlist() %>%
  unname()  
```

First define initial parameter vector for each species: log(K), log(r),
log(sigma)

``` r
# use max survey B as the initial value for K
maxsvB.GBcod <- max(index.GBcod)
ini.parms.GBcod <- c(log(maxsvB.GBcod), log(0.2), log(0.3))

maxsvB.GBherring <- max(index.GBherring)
ini.parms.GBherring <- c(log(maxsvB.GBherring), log(0.3), log(0.3))

maxsvB.GBhaddock <- max(index.GBhaddock)
ini.parms.GBhaddock <- c(log(maxsvB.GBhaddock), log(0.3), log(0.3))
```

SS model fit plots: fitted line in gray

Fit the logistic model to data (GBcod):

``` r
GBcod <- assess(harvest.GBcod,index.GBcod,calc.vcov=TRUE,ini.parms.GBcod)
GBcod
```

    ## $pars
    ## [1] -0.6587677 -0.3447666 -2.1447688
    ## 
    ## $biomass
    ##  [1] 0.5174886 0.5156808 0.5151138 0.5149240 0.5148557 0.5148295 0.5148195
    ##  [8] 0.5148161 0.5148157 0.5148167 0.5148182 0.5148197 0.5148213 0.5148226
    ## [15] 0.5148238 0.5148248 0.4775021 0.4663018 0.4626888 0.4616696 0.4614880
    ## [22] 0.4299872 0.4177530 0.4127232 0.4106655 0.4098007 0.3573837 0.3328823
    ## [29] 0.3193041 0.3107140 0.3044873 0.3664474 0.4113313 0.4375236 0.4502109
    ## [36] 0.4555832 0.4782633 0.4874881 0.4907744 0.4918679 0.4922358
    ## 
    ## $convergence
    ## [1] 0
    ## 
    ## $nll
    ## [1] -29.76121
    ## 
    ## $vcov
    ##               [,1]          [,2]          [,3]
    ## [1,]  5.142901e-04 -1.383744e-03 -1.836402e-06
    ## [2,] -1.383744e-03  6.128917e-03 -5.368664e-06
    ## [3,] -1.836402e-06 -5.368664e-06  1.219645e-02

Fit the logistic model to data (GBherring):

``` r
GBherring <- assess(harvest.GBherring,index.GBherring,calc.vcov=TRUE,ini.parms.GBherring)
GBherring
```

    ## $pars
    ## [1]  2.0329023  0.2960707 -2.5626352
    ## 
    ## $biomass
    ##  [1] 7.636217 7.634014 7.634710 7.634446 7.634527 7.634496 7.634506 7.634503
    ##  [9] 7.634504 7.634503 7.634503 7.634503 7.634503 7.634503 7.634503 7.634503
    ## [17] 6.459335 6.700269 6.736480 6.746326 6.749552 5.822593 5.835157 5.888403
    ## [25] 5.921433 5.936541 4.572963 4.314664 4.263519 4.277677 4.311208 6.083939
    ## [33] 6.823449 6.797643 6.767644 6.757800 7.366815 7.261077 7.277845 7.271290
    ## [41] 7.271597
    ## 
    ## $convergence
    ## [1] 0
    ## 
    ## $nll
    ## [1] -46.90257
    ## 
    ## $vcov
    ##               [,1]          [,2]          [,3]
    ## [1,]  1.854876e-04 -3.500361e-04  3.657208e-06
    ## [2,] -3.500361e-04  1.054879e-03 -1.120031e-05
    ## [3,]  3.657208e-06 -1.120031e-05  1.220177e-02

Fit the logistic model to data (GBhaddock):

``` r
GBhaddock <- assess(harvest.GBhaddock,index.GBhaddock,calc.vcov=TRUE,ini.parms.GBhaddock)
GBhaddock
```

    ## $pars
    ## [1]  1.8548639 -0.2489668 -2.4415102
    ## 
    ## $biomass
    ##  [1] 6.390828 6.376948 6.373826 6.373113 6.372954 6.372925 6.372930 6.372942
    ##  [9] 6.372957 6.372971 6.372985 6.372997 6.373008 6.373019 6.373028 6.373037
    ## [17] 6.283218 6.262806 6.257984 6.256780 6.256363 6.167954 6.145554 6.139379
    ## [25] 6.137258 6.136084 5.961329 5.911811 5.895651 5.888867 5.884568 6.144282
    ## [33] 6.222646 6.242604 6.247263 6.248348 6.304865 6.318769 6.322136 6.323046
    ## [41] 6.323401
    ## 
    ## $convergence
    ## [1] 0
    ## 
    ## $nll
    ## [1] -41.92663
    ## 
    ## $vcov
    ##               [,1]          [,2]          [,3]
    ## [1,]  2.974457e-04 -5.109217e-03  6.005673e-08
    ## [2,] -5.109217e-03  2.148333e-01 -2.621323e-07
    ## [3,]  6.005673e-08 -2.621323e-07  1.219581e-02

GB SS model fit plots: fitted line in gray

``` r
#model may not be using the last data year? misaligned. a hack here to fix

fit.cod <- data.frame(value= GBcod$biomass,
                      Year=c(1983:2023),
                      Species="Cod")

fit.herring <- data.frame(value= GBherring$biomass,
                       Year=c(1983:2023),
                       Species="AtlHerring")

fit.haddock <- data.frame(value= GBhaddock$biomass,
                       Year=c(1983:2023),
                       Species="Haddock")

fits <- bind_rows(fit.cod, fit.herring, fit.haddock)


GB.data %>%
  ggplot()+
  geom_point(aes(x=Year, y=ObsBio)) +
  geom_line(aes(x=Year, y=TotCatch)) +
  geom_line(data=fits, aes(x=Year, y=value), lwd=1.3, col=gray(0.5)) +
  theme_bw() +
  facet_wrap(~Species, scales = "free")
```

![](RpathOM_HCR_ConstantF_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

Single species reference points for HCRs Pars from SS models above

``` r
pars.cod <- data.frame(value= GBcod$pars,
                       Par=c("logK", "logr", "logsigma"),
                       Species="Cod")

pars.herring <- data.frame(value= GBherring$pars,
                           Par=c("logK", "logr", "logsigma"), 
                           Species="AtlHerring")

pars.haddock <- data.frame(value= GBhaddock$pars,
                           Par=c("logK", "logr", "logsigma"),
                           Species="Haddock")

pars <- bind_rows(pars.cod, pars.herring, pars.haddock) %>%
  mutate(exppar = exp(value)) 
```

Reference points from single species models above for each species.

These are going into Rpath so ok to leave on this scale.

``` r
SS.BMSY <- pars %>%
  filter(Par=="logK") %>%
  mutate(BMSY=exppar/2) %>%
  select(Species, BMSY)

# SS.Fmsy <- pars %>%
#   select(-value) %>%
#   pivot_wider(names_from = "Par", values_from = "exppar") %>%
#   mutate(Fmsy=(logr*logK)/4) %>% #so these par names are wrong because the values are exp(par)
#   select(Species, Fmsy)

SS.Fmsy <- pars %>%
  filter(Par == 'logr') %>%
  mutate(Fmsy= exppar/2) %>% 
  select(Species, Fmsy)
```

# Set-up Rsim simulation and the closed loop

## Harvest control rules

### 75% Fmsy strategy

Building on the example from
[my-first-mse](https://github.com/gavinfay/cinar-mse/blob/main/materials/exercises/day-01/my-first-mse.Rmd))
we’ll start with a simple constant F strategy.

The New England Fishery Management Council currently uses a max F for
groundfish of 75%Fmsy and for herring of 80%Fmsy. For simplicity, we
will apply 75% Fmsy to all three species (Cod, AtlHerring, Haddock) in
this control rule. Note that the Fmsy is based on single species
“assessment” using the Schaeffer model above.

This will be somewhat complicated due to the multiple fleets/species
interactions as we get further along.

``` r
control.pars <- list()

control.pars$Species <- SS.Fmsy$Species
control.pars$Htarg <- 0.75*SS.Fmsy$Fmsy #now a vector in order Cod, AtlHerring, Haddock

control <- function(estimated.biomass, control.pars, Species) {
  control.pars$Htarg[control.pars$Species==Species]
}
```

We assume perfect implementation of the strategy - in that the realized
catch is the same as the TAC. Will also have to convert the catch target
to effort as that is what drives the Rsim dynamics. Rpath standardizes
Effort to 1 so \(Catch = qEB\) simplifies to \(C = qB\) so \(q = C/B\)
or the initial \(F\).

``` r
#Cheating a little bit here
frates <- Rpath::frate.table(GB.scene)
qtable <- frates[Group %in% c('Cod', 'Haddock', 'AtlHerring'), list(Group, Gillnet, OtterTrawlLg, Midwater)]

implement <- function(TAC, q, B, ...) {
effort <- TAC / (q * B)
return(effort)
}
```

## Projection

Will project forward for 20 years

``` r
#Create initial scenario
GB.scene <- rsim.scenario(GB, GB.params, years = 1983:2042)

#Apply the same initial fishing conditions
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'OtterTrawlLg',
                            sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                            rep(5, 5), rep(10, 5),
                                                            rep(2.5, 5)))
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'Gillnet',
                           sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                           rep(5, 5), rep(10, 5),
                                                           rep(2.5, 5)))
GB.scene <- adjust.fishing(GB.scene, parameter = 'ForcedEffort', group = 'Midwater',
                           sim.year = 1983:2017, value = c(rep(0, 15), rep(2.5, 5),
                                                           rep(5, 5), rep(10, 5),
                                                           rep(2.5, 5)))

#Hopefully fixes the instability in the model
GB.scene$params$NoIntegrate[which(names(GB.scene$params$NoIntegrate) %in%
                                    c('Micronekton', 'GelZooplankton', 'Krill'))] <- 0
# det.node <- which(names(GB.scene$start_state$Biomass) == 'Detritus')
# pp.node  <- which(names(GB.scene$start_state$Biomass) == 'Phytoplankton')
# GB.scene <- adjust.forcing(GB.scene, 'ForcedBio', group = 'Detritus',
#                            sim.year = 1983:2042, 
#                            value = GB.scene$start_state$Biomass[det.node])
# GB.scene <- adjust.forcing(GB.scene, 'ForcedBio', group = 'Phytoplankton',
#                            sim.year = 1983:2042, 
#                            value = GB.scene$start_state$Biomass[pp.node])

GB.run <- rsim.run(GB.scene, years = 1983:2022, method = 'AB')
```

Run the closed loop

``` r
set.seed(123)

GB.full <- copy(GB.run)

#Divide TAC proportionally between the three gears
props <- qtable[, .SD / rowSums(.SD), .SDcols = c('Gillnet', 'OtterTrawlLg',
                                                  'Midwater')]
props[, Group := qtable[, Group]]

for(iyear in 2023:2042){
  GB.full <- rsim.step(GB.scene, GB.full, method = 'AB', iyear)
    
  #New observations
  new.cod <- GenData(GB.full, 'Cod', iyear, Sigma = 0.1)
  new.had <- GenData(GB.full, 'Haddock', iyear, Sigma = 0.1)
  new.her <- GenData(GB.full, 'AtlHerring', iyear, Sigma = 0.1)
  
  #New TAC
  index.cod <- new.cod$ObsBio
  index.had <- new.had$ObsBio
  index.her <- new.her$ObsBio
  
  #check for correct order of control.pars vector which goes Cod, AtlHerring, Haddock
  TAC.cod <- control(index.cod, control.pars, "Cod") * index.cod
  TAC.had <- control(index.had, control.pars, "Haddock") * index.had
  TAC.her <- control(index.her, control.pars, "AtlHerring") * index.her
  
  #proportion TAC
  TAC.cod.lgm <- TAC.cod * props[Group == 'Cod', OtterTrawlLg]
  TAC.cod.gil <- TAC.cod * props[Group == 'Cod', Gillnet]
  TAC.cod.mid <- TAC.cod * props[Group == 'Cod', Midwater]
  
  TAC.had.lgm <- TAC.had * props[Group == 'Haddock', OtterTrawlLg]
  TAC.had.gil <- TAC.had * props[Group == 'Haddock', Gillnet]
  TAC.had.mid <- TAC.had * props[Group == 'Haddock', Midwater]
  
  TAC.her.lgm <- TAC.her * props[Group == 'AtlHerring', OtterTrawlLg]
  TAC.her.gil <- TAC.her * props[Group == 'AtlHerring', Gillnet]
  TAC.her.mid <- TAC.her * props[Group == 'AtlHerring', Midwater]
  
  #Implement new TAC via effort
  #Large mesh trawl and gillnet will be set by the lower effort between cod and haddock
  effort.lgm <- min(c(implement(TAC.cod.lgm, qtable[Group == 'Cod', OtterTrawlLg],
                                index.cod),
                      implement(TAC.had.lgm, qtable[Group == 'Haddock', OtterTrawlLg],
                                index.had)))
  effort.gil <- min(c(implement(TAC.cod.gil, qtable[Group == 'Cod', Gillnet],
                                index.cod),
                      implement(TAC.had.gil, qtable[Group == 'Haddock', Gillnet],
                                index.had))) 
  #Midwater set by herring
  effort.mid <- implement(TAC.her.mid, qtable[Group == 'AtlHerring', Midwater],
                          index.her)
  
  #Set new Effort
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'OtterTrawlLg', 
                             sim.year = iyear + 1, value = effort.lgm)
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'Gillnet', 
                             sim.year = iyear + 1, value = effort.gil)
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'Midwater', 
                             sim.year = iyear + 1, value = effort.mid)
  
  } 
```

``` r
#Plot results
GB.bio <- as.data.table(GB.full$out_Biomass)
GB.bio[, Month := 1:nrow(GB.bio)]
GB.bio <- data.table::melt(GB.bio, id.vars = 'Month', variable.name = 'Group', 
                           value.name = 'Biomass')
start <- data.table::as.data.table(GB.full$start_state$Biomass)
start[, Group := names(GB.full$start_state$Biomass)]

GB.bio <- merge(GB.bio, start, by = 'Group')
GB.bio[, Rel.biomass := Biomass / V1]

ggplot(GB.bio[Group %in% c('Cod', 'Haddock', 'AtlHerring')],
       aes(x = Month, y = Rel.biomass, col = Group)) +
  geom_line()
```

![](RpathOM_HCR_ConstantF_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

Collect the results for further analysis

``` r
#Get in a similar format to SS models run in other rmds
sp.cols <- which(colnames(GB.full$annual_Biomass) %in% c('Cod', 'Haddock', 'AtlHerring'))

#Biomass
biomass <- as.data.table(GB.full$annual_Biomass[, sp.cols])
biomass[, Year := 1983:2042]
biomass <- data.table::melt(biomass, id.vars = 'Year', variable.name = 'Species')
biomass[, type := 'biomass']

#Catch
catch <- as.data.table(GB.full$annual_Catch[, sp.cols])
catch[, Year := 1983:2042]
catch <- data.table::melt(catch, id.vars = 'Year', variable.name = 'Species')
catch[, type := 'catch']

output <- rbindlist(list(biomass, catch))
output[, HCR := 'ConstantF_min']
```

# Rerun simulation but with max effort taken rather than min

``` r
set.seed(123)

GB.full <- copy(GB.run)

#Divide TAC proportionally between the three gears
props <- qtable[, .SD / rowSums(.SD), .SDcols = c('Gillnet', 'OtterTrawlLg',
                                                  'Midwater')]
props[, Group := qtable[, Group]]

for(iyear in 2023:2042){
  GB.full <- rsim.step(GB.scene, GB.full, method = 'AB', iyear)
    
  #New observations
  new.cod <- GenData(GB.full, 'Cod', iyear, Sigma = 0.1)
  new.had <- GenData(GB.full, 'Haddock', iyear, Sigma = 0.1)
  new.her <- GenData(GB.full, 'AtlHerring', iyear, Sigma = 0.1)
  
  #New TAC
  index.cod <- new.cod$ObsBio
  index.had <- new.had$ObsBio
  index.her <- new.her$ObsBio
  
  #check for correct order of control.pars vector which goes Cod, AtlHerring, Haddock
  TAC.cod <- control(index.cod, control.pars, "Cod") * index.cod
  TAC.had <- control(index.had, control.pars, "Haddock") * index.had
  TAC.her <- control(index.her, control.pars, "AtlHerring") * index.her
  
  #proportion TAC
  TAC.cod.lgm <- TAC.cod * props[Group == 'Cod', OtterTrawlLg]
  TAC.cod.gil <- TAC.cod * props[Group == 'Cod', Gillnet]
  TAC.cod.mid <- TAC.cod * props[Group == 'Cod', Midwater]
  
  TAC.had.lgm <- TAC.had * props[Group == 'Haddock', OtterTrawlLg]
  TAC.had.gil <- TAC.had * props[Group == 'Haddock', Gillnet]
  TAC.had.mid <- TAC.had * props[Group == 'Haddock', Midwater]
  
  TAC.her.lgm <- TAC.her * props[Group == 'AtlHerring', OtterTrawlLg]
  TAC.her.gil <- TAC.her * props[Group == 'AtlHerring', Gillnet]
  TAC.her.mid <- TAC.her * props[Group == 'AtlHerring', Midwater]
  
  #Implement new TAC via effort
  #Large mesh trawl and gillnet will be set by the lower effort between cod and haddock
  effort.lgm <- max(c(implement(TAC.cod.lgm, qtable[Group == 'Cod', OtterTrawlLg],
                                index.cod),
                      implement(TAC.had.lgm, qtable[Group == 'Haddock', OtterTrawlLg],
                                index.had)))
  effort.gil <- max(c(implement(TAC.cod.gil, qtable[Group == 'Cod', Gillnet],
                                index.cod),
                      implement(TAC.had.gil, qtable[Group == 'Haddock', Gillnet],
                                index.had))) 
  #Midwater set by herring
  effort.mid <- implement(TAC.her.mid, qtable[Group == 'AtlHerring', Midwater],
                          index.her)
  
  #Set new Effort
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'OtterTrawlLg', 
                             sim.year = iyear + 1, value = effort.lgm)
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'Gillnet', 
                             sim.year = iyear + 1, value = effort.gil)
  GB.scene <- adjust.fishing(GB.scene, 'ForcedEffort', 'Midwater', 
                             sim.year = iyear + 1, value = effort.mid)
  
  } 
```

``` r
#Plot results
GB.bio <- as.data.table(GB.full$out_Biomass)
GB.bio[, Month := 1:nrow(GB.bio)]
GB.bio <- data.table::melt(GB.bio, id.vars = 'Month', variable.name = 'Group', 
                           value.name = 'Biomass')
start <- data.table::as.data.table(GB.full$start_state$Biomass)
start[, Group := names(GB.full$start_state$Biomass)]

GB.bio <- merge(GB.bio, start, by = 'Group')
GB.bio[, Rel.biomass := Biomass / V1]

ggplot(GB.bio[Group %in% c('Cod', 'Haddock', 'AtlHerring')],
       aes(x = Month, y = Rel.biomass, col = Group)) +
  geom_line()
```

![](RpathOM_HCR_ConstantF_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

Collect the results for further analysis

``` r
#Get in a similar format to SS models run in other rmds
sp.cols <- which(colnames(GB.full$annual_Biomass) %in% c('Cod', 'Haddock', 'AtlHerring'))

#Biomass
biomass <- as.data.table(GB.full$annual_Biomass[, sp.cols])
biomass[, Year := 1983:2042]
biomass <- data.table::melt(biomass, id.vars = 'Year', variable.name = 'Species')
biomass[, type := 'biomass']

#Catch
catch <- as.data.table(GB.full$annual_Catch[, sp.cols])
catch[, Year := 1983:2042]
catch <- data.table::melt(catch, id.vars = 'Year', variable.name = 'Species')
catch[, type := 'catch']

output2 <- rbindlist(list(biomass, catch))
output2[, HCR := 'ConstantF_max']

output <- rbindlist(list(output, output2))

saveRDS(output, file = here('multispecies', 'data', 'MS_ConstantF.rds'))
```
