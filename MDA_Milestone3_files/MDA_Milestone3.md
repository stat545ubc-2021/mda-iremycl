Mini Data-Analysis Deliverable 3
================

# Setup

Begin by loading your data and the tidyverse package below:

``` r
library(datateachr) # <- might contain the data you picked!
library(tidyverse)
library(broom)
```

From Milestone 2, you chose two research questions. What were they? Put
them here.

<!-------------------------- Start your work below ---------------------------->

1.  *Is there a correlation between the building contractor and project
    value? Would that indicate the budget of the contractor?*
2.  *Is there a relationship between project value and property use? eg.
    Does it cost more to build offices or houses?*
    <!----------------------------------------------------------------------------->

# Exercise 1: Special Data Types

I had a plot in my previous milestone for Question 1.

``` r
building_permits[building_permits == ""] <- NA


top_contract <- building_permits %>%
  drop_na(building_contractor) %>%
  add_count(building_contractor) %>%
  filter(n > 50) %>%
  select(-n) 

summary_top_contract = top_contract %>%
  group_by(building_contractor) %>%
  summarize(mean_projval = mean(project_value, na.rm = T),
            #range_projval = range(project_value, na.rm = T), printing min and max as it looks better
            median_projval = median(project_value, na.rm = T),
            min_projval = min(project_value, na.rm = T),
            max_projval = max(project_value, na.rm = T),
            sd_projval = sd(project_value, na.rm = T),
            num_proj = n()
            )

ggplot(summary_top_contract, 
       aes(x = log(mean_projval), #Taking log of project values because of skewness towards a large value (One point much larger than the rest of the bulk data)
           y = reorder(building_contractor, -mean_projval), #Ordering the contractors based on their project values)
           color= building_contractor)) +
  geom_point() +
  theme(legend.position = "none") +
  labs(title = "Log Mean Project Value of Building Contractors",
       x = "Log Mean Project Value",
       y = "Building Contractors")
```

![](MDA_Milestone3_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

## Tasks

**1**: Produce a new plot that reorders a factor in your original plot,
using the `forcats` package (3 points). Then, in a sentence or two,
briefly explain why you chose this ordering (1 point here for
demonstrating understanding of the reordering, and 1 point for
demonstrating some justification for the reordering, which could be
subtle or speculative.)

``` r
plot_reordered <- ggplot(summary_top_contract, 
       aes(x = log(mean_projval), #Taking log of project values because of skewness towards a large value (One point much larger than the rest of the bulk data)
           y = forcats::fct_reorder(building_contractor, mean_projval, .desc = T), #I modified this section to use fct_reorder from forcats package.
           color= building_contractor)) +
  geom_point() +
  theme(legend.position = "none") +
  labs(title = "Log Mean Project Value of Building Contractors",
       x = "Log Mean Project Value",
       y = "Building Contractors")

plot_reordered
```

![](MDA_Milestone3_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

Here, I choose to reorder the building contractors based on their (log)
mean project value on decreasing order. After reordering, it is easier
to compare the contractors and see the ones with minimum and maximum
project value, and how the project values are distributed ammong
contractors.

**2**: Produce a new plot that groups some factor levels together into
an “other” category (or something similar), using the `forcats` package
(3 points). Then, in a sentence or two, briefly explain why you chose
this grouping (1 point here for demonstrating understanding of the
grouping, and 1 point for demonstrating some justification for the
grouping, which could be subtle or speculative.)

``` r
summary_top_contract %>%
  mutate(class = forcats::fct_collapse(summary_top_contract$building_contractor, 
                                        "High" = c(summary_top_contract$building_contractor[log(summary_top_contract$mean_projval) > 13]), 
                                        "Medium" = c(summary_top_contract$building_contractor[log(summary_top_contract$mean_projval) < 13]),
                                        "Low" = c(summary_top_contract$building_contractor[log(summary_top_contract$mean_projval) < 11]))) %>%
  ggplot(aes(x = forcats::fct_infreq(class))) +
  geom_bar(color = 'black', fill = 'firebrick') + 
  labs(title = "Number of Building Contractors in Each Budget Class",
       x = "Contractor Class",
       y = "Count")
```

![](MDA_Milestone3_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

Here, I want to group the contractors based on their (log) mean project
value, with levels “high”, “medium” and “low”. For this, I created a new
column “Budget” in summary\_top\_contract table using `fct_collapse()`
function from forcats package. After this, to see the number of
contractors in each class, so I plotted the histogram using `fct_infreq`
function.

# Exercise 2: Modelling

## 2.0

Pick a research question, and pick a variable of interest (we’ll call it
“Y”) that’s relevant to the research question. Indicate these.

**Research Question**: *2. Is there a relationship between project value
and property use? eg. Does it cost more to build offices or houses?*

**Variable of interest**: *Project value*

## 2.1

Fit a model or run a hypothesis test that provides insight on this
variable with respect to the research question. Store the model object
as a variable, and print its output to screen.

``` r
##Getting the tidy data from previous Milestone:

building_permits2 <- building_permits %>%
  separate(address, c("adress", "PostalCode"), sep = "BC", remove = FALSE) %>% # create a new variable named PostalCode, separating the address column by BC
  select(-adress) #drop the other new created column

building_permits2[building_permits2 == ""] <- NA #Some of the addresses do not include Postal Code, so this column is empty in some cells, so here, I fill these with NA.

#Now selecting projects with PostalCode that has 30 or more buildings
top_postalcode <- building_permits2  %>%
  drop_na(PostalCode) %>%
  add_count(PostalCode) %>% #Computing number of observations
  filter(n > 30) %>%
  mutate(FSA = substr(PostalCode,1,4))


sub_top_postalcode = top_postalcode %>% 
  group_by(property_use) %>%
  drop_na(property_use) %>%
  summarise(n= n()) %>%
  arrange(desc(n)) %>%
  top_n(5) 
```

    ## Selecting by n

``` r
sub_top_postalcode = as_vector(sub_top_postalcode$property_use)

top_postalcode = top_postalcode %>%
  filter(property_use %in% sub_top_postalcode)%>%
  mutate(logPropVal = log10(project_value + 1)) %>%
  drop_na(property_use)


model <- lm(logPropVal ~ property_use, data = top_postalcode)
broom::tidy(model)
```

    ## # A tibble: 5 × 5
    ##   term                           estimate std.error statistic  p.value
    ##   <chr>                             <dbl>     <dbl>     <dbl>    <dbl>
    ## 1 (Intercept)                       4.39     0.0601    73.0   0       
    ## 2 property_useInstitutional Uses    0.701    0.352      1.99  4.69e- 2
    ## 3 property_useOffice Uses           0.616    0.0990     6.23  6.16e-10
    ## 4 property_useRetail Uses           0.550    0.154      3.58  3.54e- 4
    ## 5 property_useService Uses          0.208    0.294      0.707 4.80e- 1

## 2.2

Produce something relevant from your fitted model: either predictions on
Y, or a single value like a regression coefficient or a p-value.

``` r
#I will use broom::tidy function to summarize the results from ANOVA model. Then I will print the result as a tibble. We are interested in the p-value column, which indicates if the mean is different between groups.

summarymodel <- broom::tidy(model)
summarymodel
```

    ## # A tibble: 5 × 5
    ##   term                           estimate std.error statistic  p.value
    ##   <chr>                             <dbl>     <dbl>     <dbl>    <dbl>
    ## 1 (Intercept)                       4.39     0.0601    73.0   0       
    ## 2 property_useInstitutional Uses    0.701    0.352      1.99  4.69e- 2
    ## 3 property_useOffice Uses           0.616    0.0990     6.23  6.16e-10
    ## 4 property_useRetail Uses           0.550    0.154      3.58  3.54e- 4
    ## 5 property_useService Uses          0.208    0.294      0.707 4.80e- 1

From this table, we can see that the p-values are significant for
Institutional, Office and Retail Uses, which means their (log) mean
property values are different than the intercept.

# Exercise 3: Reading and writing data

Get set up for this exercise by making a folder called `output` in the
top level of your project folder / repository. You’ll be saving things
there.

## 3.1

Take a summary table that you made from Milestone 2 (Exercise 1.2), and
write it as a csv file in your `output` folder. Use the `here::here()`
function.

  - **Robustness criteria**: You should be able to move your Mini
    Project repository / project folder to some other location on your
    computer, or move this very Rmd file to another location within your
    project repository / folder, and your code should still work.
  - **Reproducibility criteria**: You should be able to delete the csv
    file, and remake it simply by knitting this Rmd file.

<!-- end list -->

``` r
write_csv(top_postalcode, file = here::here("output","top_postalcodeforMDA3.csv"))
```

## 3.2

Write your model object from Exercise 2 to an R binary file (an RDS),
and load it again. Be sure to save the binary file in your `output`
folder. Use the functions `saveRDS()` and `readRDS()`.

  - The same robustness and reproducibility criteria as in 3.1 apply
    here.

<!-- end list -->

``` r
#Saving the model
saveRDS(object = model, file = here::here("output","anova_model.rds"))

#Reloading the model
anova_model2 = readRDS(here::here("output","anova_model.rds"))
tidy(anova_model2) #Check it was loaded correctly
```

    ## # A tibble: 5 × 5
    ##   term                           estimate std.error statistic  p.value
    ##   <chr>                             <dbl>     <dbl>     <dbl>    <dbl>
    ## 1 (Intercept)                       4.39     0.0601    73.0   0       
    ## 2 property_useInstitutional Uses    0.701    0.352      1.99  4.69e- 2
    ## 3 property_useOffice Uses           0.616    0.0990     6.23  6.16e-10
    ## 4 property_useRetail Uses           0.550    0.154      3.58  3.54e- 4
    ## 5 property_useService Uses          0.208    0.294      0.707 4.80e- 1
