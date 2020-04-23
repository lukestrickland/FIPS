# Fatigue Impairment Prediction Suite (FIPS)

<img align="right" src="inst/logo/FIPS_logo.png?raw=true" alt="FIPSLOGO" width="200"/> 

FIPS provides researchers and practitioners comprehensive set of functions for applying bio-mathematical models (BMMs) of fatigue. FIPS is a young project under active development and is implemented in the R programming language. 

FIPS includes a set of well documented functions for transforming sleep and actigraphy data to the data frame structure (called a `FIPS_df`) required for executing BMM simulations. Importantly, FIPS includes a set of functions for simulating from and interpreting several forms of BMM, including the Unified Model and Three Process Model. All models are extendable and include customisable parameters. 

## What are BMMs?

BMMs are a class of biological phenomenological models which are used to predict the neuro-behavioural outcomes of fatigue (e.g., alertness, performance) using sleep-wake history. There are several different BMM implementations, but most have their roots in Borbély's (1982) two process model which stipulates that sleepiness/performance impairment functions in response to two processes: a circadian process and a homeostatic process. BMMs enable hypothesis testing of the latent factors underlying the relationships between sleep, fatigue, and human performance. For example, they enable researchers to estimate the relative contributions of homeostatic processes on fatigue, relative to endogenous circadian processes. These models are also frequently applied by defence and industrial sectors to support system safety as part of broader fatigue management strategies. FIPS is the first open-source BMM framework enabling practitioners to inspect, validate, and ideally extend BMMs. 

# Using FIPS

## Installation
To install the latest development version of FIPS:

```R
# install.packages('remotes') # if remotes not installed
remotes::install_github("humanfactors/FIPS")
```
We highly recommended building the vignettes (with `build_vignettes = TRUE`; though note this requires additional dependencies) as both vignettes offer comprehensive walkthroughs of the core features of the package .

## Simple Use Case

Full walkthroughs for using FIPS can be found in the vignettes: `vignette("plotting","FIPS")` and `vignette("generation-and-three-process-simulation","FIPS")` or by accessing the source Rmarkdown files under `./vignettes`.

Currently all FIPS simulations start with the *Sleep Data Format*, which is shown below. Importantly, the datetimes in the columns must be POSIXct datetime objects. We suggest leveraging lubridate for a simple datetime interface. Ideally, you should use these column names and format to avoid conflicts with the generated column names. 

**The Sleep Data Format** looks as follows:

|sleep.start         |sleep.end           | sleep.id|
|:-------------------|:-------------------|--------:|
|2018-05-11 07:07:00 |2018-05-11 12:55:00 |        1|
|2018-05-12 06:14:00 |2018-05-12 12:50:00 |        2|
|2018-05-14 01:55:00 |2018-05-14 06:29:00 |        3|
|2018-05-14 13:30:00 |2018-05-14 14:58:00 |        4|

For testing purposes, the following snippet shows how to generate a FIPS compliant sleep data frame.

```{r}
example.sleeptimes <- tibble::tibble(
  sleep.start = seq(
    from = lubridate::ymd_hms('2018-05-01 23:00:00', tz = "Australia/Perth"), 
    to = lubridate::ymd_hms('2018-05-07 17:00:00', tz = "Australia/Perth"),
    by = '24 hours'),
  sleep.end = sleep.start + lubridate::dhours(7.5),
  sleep.id = rank(sleep.start))
```
Advanced users may be able to create a dataframe compliant with the *FIPS data format* by examining `help("FIPS_df")`. There are plans to offer additional methods for transforming data types to the `FIPS_df` format (e.g., binary sequence with equidistance temporal spacing).

Now that you have the correct sleep times input structure, you can generate the `FIPS_df` from the sleep data format via the `parse_sleeptimes` function.

```R
# Simulation start date time (i.e., when you want first predictions to start)
simulation.start = lubridate::ymd_hms('2018-05-01 07:00:00', tz = "Australia/Perth")
# Simulation end date time (i.e., when you want predictions to end)
simulation.end = lubridate::ymd_hms('2018-05-07 21:00:00', tz = "Australia/Perth")
# The Continuous FIPS_df dataframe format
# This creates the format ready for simulation
simulated.dataframe = parse_sleeptimes(
  sleeptimes = example.sleeptimes,
  series.start = simulation.start,
  series.end = simulation.end,
  sleep.start.col = "sleep.start",
  sleep.end.col = "sleep.end",
  sleep.id.col = "sleep.id",
  roundvalue = 5
  )

```
The resulting dataframe output (of class `FIPS_df`) can then be sent the simulation dispatch function `FIPS_simulate()`. The simulation functions require a parameter vector/list (pvec). Please see help for `TPM_make_pvec()` or `unified_make_pvec()` for more information on generating these vectors.

In the example below, we will run a Three Process Model simulation over the `FIPS_df` series we just created. To do this, we will use the `FIPS_simulation` function, which takes in three arguments: a `FIPS_df` object, a specification of a `modeltype` (see help for model types currently implemented), and a `pvec` which is a vector of parameters for the model.

Calling `FIPS_simulate()` will the produces a `FIPS_df` with model predictions and predicted model parameter values (e.g., `s`, `c`). The returned `FIPS_df` will also now inherit the `FIPS_simulation` class.  A `FIPS_simulation` object has attributes containing the parameter vector, the `modeltype` string, and the `pvec` used, and several other values used internally. A custom print function of the object will reveal all this information. Note that this print function does mask some of the columns for ease of reading.

```r
# Run a simulation with the three process model
TPM.simulation.results = FIPS_simulate(
  FIPS_df = simulated.dataframe, # The FIPS_df
  modeltype = "TPM",             # three process model
  pvec = TPM_make_pvec()       # parameter vector with defaults
  )
```

You now can access printing, summary and plot methods for the FIPS_simulation object. Note that further transformations to the object in dplyr and similar Tidyverse packages will remove attributes.

```R
plot(TPM.simulation.results)
summary(TPM.simulation.results)
print(TPM.simulation.results)
```

## Notes on Design Choices
As of version 0.1.0, parts of the interface should be considered relatively unstable (i.e., subject to future change). A number of difficult decisions had to be made regarding the design of FIPS objects and models which we acknowledge are not "perfect". To understand the challenges, it is important to understand how dynamic models are processed analytically.

BMMs can be best described as "dynamic process models". This class of models require the calculation of _time varying process variables_ prior to model fitting or simulation. For instance, the functions to caclulate the `S` (homeostatic) parameter require information on how long the individual has been awake or asleep at that time. Moreover, the input to `S` functions also require knowledge of the past states of `S`, such that these operations cannot be trivially vectorised (if at all).

This requires rigid data structures in formats R is not particularly suitable for. As such, some design choices in the package may seem inflexible (e.g., requirement to start with sleep times data structure), but are in place to ensure consistency and reliability in the code. It would be extremely challenging to develop validators for arbitrary user generated data structures (as can be done in `lme4` and `brms`). Doing so would could also greatly increase the complexity of the API (as users would need to need to specify all variables used for calculations). However, a future goal of the project will be to iteratively allow more input formats (so if you have any requests please file an issue).

# Additional Terms for Academic Usage
In addition to the rights stipulated in the GNU Affero GPL-3, we request that all work leveraging FIPS provide a direct citation to the software package. Please contact the authors for a citation (as of 22/04/2020). We aim to have a manuscript for citation soon.

# Authors

- Dr. Michael David Wilson (Future of Work Institute ― Curtin University)
- Dr. Luke Strickland (Future of Work Institute ― Curtin University)
- Dr. Timothy Ballard (University of Queensland)
