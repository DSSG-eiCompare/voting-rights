---
title: "Bayesian Improved Surname Geocoding (BISG)"
author: "null"
date: "8/5/2020"
output:
  html_document: default
  pdf_document: default
---
This vignette demonstrates how to perform Bayesian Improved Surname Geocoding when the race/ethncity of individuals are unknown within a dataset.

## *What is Bayesian Improved Surname Geocoding?*

Bayesian Improved Surname Geocoding (BISG) is a method that applies the Bayes Rule/Theorem to predict the race/ethnicity of an individual using the individual's surname and geocoded location [Elliott et. al 2008, Elliot et al. 2009, Imai and Khanna 2016]. 

Specifically, BISG first calculates the prior probability of **i** individual being of a ceratin **r** racial group given their **s** surname, or p(r<sub>i<sub>|s<sub>i<sub>). The prior probability created from the surname is then updated with the probability of the **i** individual living in a **g** geographic location belonging to a *r* racial group, or p(g<sub>i<sub>|r<sub>i<sub>). The following equation describes how BISG calculates race/ethnicity of individuals using Bayes Theorem, given the surname and geographic location, and specifically when race/ethncicty is unknown :
      

      ![BISG Equation for Predicting Race/Ethnicity.]/github/eiCompare/vignettes/bisg_equation.PNG

In R, the package that performs BISG is called, WRU: Who Are You `wru` [cite WRU package]. This vignette will walk you through how to prepare your geocoded voter file for performing BISG by stepping you throught the processing of cleaning your voter file, prepping voter data for running the BISG, and finally, performing BISG to obtain racial/ethnic probailities of individuals in a voter file.

## *Performing BISG on your data*
We will perform BISG using the previous Gwinnett and Fulton county voter registration data called `gwin_fulton_5k.csv` that was geocoded in the **eiCompare: Geocoding vignette**. 

The first step in performing BISG is to geocode your voter file addresses. For information on geocoding, visit the Geocoding Vignette. 

Let's begin by loading your geocoded voter data into R/RStudio.

### Step 1: Load R libraries/packages, voter file, and census data

Load the R packages needed to perform BISG. If you have not already downloaded the following packages, please install these packages.
```{r}
# Load libraries/packages

suppressMessages(c(
  library(devtools),
  library(tidyverse),
  library(stringr),
  library(tigris),
  library(leaflet),
  library(sf),
  library(eiCompare),
  library(wru),
  library(readr)
))
```

```{r}
# source files
source("~/github/eiCompare/R/wru_predict_race_wrapper.R")
source("~/github/eiCompare/R/voter_file_utils.R")
```

```{r}
path_census <- "~/shared/georgia/"
path <- "~/github/eiCompare/data/"
```

Load in census data, the shape_file and geocoded voter registation data with latitude and longitude coordinates Gwinnett and Fulton .

Make sure to load your census data that details certain geographies (i.e. counties, cities, tracts, blocks, etc.)
```{r}
# Load Georgia census data
census_data <- readRDS(paste(path_census, "georgia_census.rds", sep = ""))
```

Next, load the state shape file using the sf::st_read() function.
```{r}
# Load Georgia block shape file
shape_file <- blocks(state = "GA", county = c("Gwinnett","Fulton"))
```

Load geocoded voter file.
```{r}
# Load geocoded voter registration file
voter_file_geocoded <- read_csv(paste(path, "ga_geo.csv", sep=""))
```

Obtain the first six rows of the voter file to check that the file has downloaded properly.
```{r}
# Check the first six rows of the voter file
head(voter_file_geocoded, 6)
```

View the column names of the voter file. Some of these columns will be used along the journey to performing BISG.
```{r}
# Find out names of columns in voter file
names(voter_file_geocoded)
```

Check the dimensions (the number of rows and columns) of the voter file.
```{r}
# Get the dimensions of the voter file
dim(voter_file_geocoded)
```
There are 4440 voters (or observations) and 45 columns in the voter file.


### Step 2: De-duplicate the voter file.

The next step involves removing duplicate voter IDs from the voter file, using the `dedupe_voter_file` function. 
```{r}
# Rename registration_id to voter_id
names(voter_file_geocoded)[names(voter_file_geocoded) == "registration_number"] <- "voter_id"

# Separate latitude and longitude into columns
voter_file_geocoded <- voter_file_geocoded %>%
  extract(geometry, c("lon", "lat"), "\\((.*), (.*)\\)", convert = TRUE)
```

```{r}
# Check column names for lat and lon columns
names(voter_file_geocoded)
```

```{r}
# Remove duplicate voter IDs (the unique identifier for each voter)
voter_file_dedupe <- dedupe_voter_file(voter_file = voter_file_geocoded, voter_id = "voter_id")

# Check new dimensions of voter file after removing duplicate voter IDs
dim(voter_file_dedupe)
names(voter_file_dedupe)
```

There are no duplicate voter IDs in the dataset.

### Step 5: Perform BISG and obtain the predicted race/ethnicity of each voter.
```{r}
# Convert the voter_shaped_merged file into a data frame for performing BISG.
voter_file_complete <- as.data.frame(voter_file_dedupe)
class(voter_file_complete)
```

```{r}
# Perform BISG
bisg_file <- eiCompare::wru_predict_race_wrapper(
  voter_file = voter_file_complete,
  census_data = census_data,
  voter_id = "voter_id",
  surname = "last_name",
  state = "GA",
  county = "COUNTYFP10",
  tract = "TRACTCE10",
  block = "BLOCKCE10",
  census_geo = "block",
  use_surname = TRUE,
  surname_only = FALSE,
  surname_year = 2010,
  use_age = FALSE,
  use_sex = FALSE,
  return_surname_flag = TRUE,
  return_geocode_flag = TRUE,
  verbose = TRUE
)
```

```{r}
#Check the object for BISG output
head(bisg_file$bisg)

#Assign BISG list data frame, bisg$bisg, to bisg_tbl
bisg_tbl <-bisg_file$bisg

```

```{r}
# Save file as a .csv
write_csv(bisg_tbl, paste(path, "ga_geo_output.csv"))
```


## Summarizing BISG output
```{r}
summary(bisg_tbl)
```


```{r}
# Obtain aggregate values for the BISG results by county
bisg_agg <- precinct_agg_combine(voter_file = bisg_tbl,
                                        group_col = "county",
                                        race_cols = c("pre.whi", "pred.bla", "pred.his", "pred.asi", "pred.oth"),
                                        race_keys = c("White", "Black", "Hispanic", "Asian", "Other"),
                                        include_total = FALSE)

bisg_agg

```

```{r}
#Assign values to county names for barplot
bisg_agg$county[bisg_agg$county == "121"] <- "Fulton"
bisg_agg$county[bisg_agg$county == "135"] <- "Gwinnett"

bisg_agg
```

### Barplot of BISG results
```{r}
bisg_bar <- bisg_agg %>%
    gather("Type", "Value",-county) %>%
    ggplot(aes(county, Value, fill = Type)) +
    geom_bar(position = "dodge", stat = "identity") +
    labs(title="BISG Predictions for Fulton and Gwinnett Counties", y="Proportion", x="Counties") +
    theme_bw()

bisg_bar + scale_color_discrete(name="Race/Ethnicity Proportions")
```

### Choropleth Map
Finally, we will map the BISG data onto choropleth maps.

```{r}
names(bisg_tbl)
```

```{r}
bisg_df <-bisg_tbl %>%
  select(block, pred.whi, pred.bla, pred.his, pred.asi, pred.oth)

bisg_df
```

```{r}
names(bisg_df)[names(bisg_df)=="block"] <- "BLOCKCE10"
bisg_sf <- left_join(shape_file, bisg_df, by="BLOCKCE10")

ggplot(data = bisg_sf) +
  geom_sf(aes(fill = 'pred.bla'))
```
