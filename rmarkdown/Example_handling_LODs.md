---
title: "Example of how to deal with no detects"
author: "Jose Luis Rodriguez Gil"
date: "05/11/2020"
output: 
  html_document:
    keep_md: true
---





# Making some mock data to work with

lets make a table of 100 samples with values for two compounds


```r
lod_data <- tibble(sample_id = seq(1,100),                                  # Just a sequence of numbers from 1 to 100 as sample ID
                   compound_a = rnorm(100, 24, 20),                         # 100 concentrations, normally distributed around 24 with an SD of 20
                   compound_b = rnorm(100, 50, 50)) %>%                     # 100 concentrations, normally distributed around 50 with an SD of 50
  mutate(compound_a = ifelse(compound_a < 10, "<10.0", compound_a),
         compound_b = ifelse(compound_b < 20.5, "<20.5", compound_b))       # no we use ifelse() to give all samples below the LOD an "<LOD" value

print(lod_data)
```

```
## # A tibble: 100 x 3
##    sample_id compound_a       compound_b      
##        <int> <chr>            <chr>           
##  1         1 18.2536257604069 128.650881415697
##  2         2 31.3117637522976 43.2665229978174
##  3         3 26.3594414454959 21.4488660477753
##  4         4 <10.0            <20.5           
##  5         5 38.0273324774875 38.2310119424659
##  6         6 71.9456833945761 <20.5           
##  7         7 35.746663471494  47.0092896554522
##  8         8 19.6314143114086 28.4785262017548
##  9         9 24.8808402407629 31.7743010516316
## 10        10 27.4458081630701 109.67303574835 
## # … with 90 more rows
```

# change the <LODs for zeros

One aproach that can be followed is to change the samples below the Limit od Detection (LOD) for a zero:


```r
lod_data %>% 
  pivot_longer(cols = -sample_id, names_to = "compound", values_to = "concentration") %>% 
  mutate(concentration = str_replace(concentration, "^<[:graph:]*", "0")) %>% 
  mutate(concentration = as.numeric(concentration))
```

```
## # A tibble: 200 x 3
##    sample_id compound   concentration
##        <int> <chr>              <dbl>
##  1         1 compound_a          18.3
##  2         1 compound_b         129. 
##  3         2 compound_a          31.3
##  4         2 compound_b          43.3
##  5         3 compound_a          26.4
##  6         3 compound_b          21.4
##  7         4 compound_a           0  
##  8         4 compound_b           0  
##  9         5 compound_a          38.0
## 10         5 compound_b          38.2
## # … with 190 more rows
```

`str_replace()` from the `{stringr}` package (loads automatically with `{tydiverse}`) is quite handy for this, it looks for whatever pattern you tell it, and changes it for whatever you say.

In this case we use **regular expressions** (regex) to fin a string of any length (the `*` at the end) of any content (numbers, letters, symbols) (the `[:graph:]` part), that starts with "<" (the `^<` part). When it finds these, it change sthem to zeros

Because there were "<" in the original data, the column was loaded as text, so the last step s to turn it into a numeric variable.

I recomend checking the `{stringr}` [cheatsheet](https://github.com/rstudio/cheatsheets/blob/master/strings.pdf), for more detail on **regex**. They can be quite a pain.

# Change <LODs to LOD/2

Another common substitution aproach is to change the samples below the lod for a value of 1/2 the LOD. This aproach assumes that all those samples would have their own normal distribution with a min of 0 and a max of the LOD, so the mean should be LOD/2.

The ideal case scenario would be for you to have a separate table of LODs. So you could join this with your results table and have a complete LOD column that we can use to make our calculations. Most times, this is not the case, so the most convenient aproach would be to extract the info contained in those "<XX" samples to create an LOD column, which then we will use to mutate the concentration column with a LOD/2 value


```r
lod_data %>% 
  pivot_longer(cols = -sample_id, names_to = "compound", values_to = "concentration") %>% 
    mutate(lod = str_extract(concentration, "(?<=<)[:graph:]*")) %>%    # here we use the "preceeded by <" ("(?<=<)") aproach, as we only want to keep the numbers
  mutate(lod = as.numeric(lod)) %>%  # we need to calculate the LOD/2, so we need it to be a number
  mutate(concentration = str_replace(concentration, "^<[:graph:]*", as.character(lod/2))) %>%  # same as before but now we replace with LOD/2. Additional "problem", because here we are working with strings, if we try to give it a number, it doesnt like it, so we need to soround that LOD/2 by that "as.character()"
  mutate(concentration = as.numeric(concentration)) # Now it can all be converted to numbers again!
```

```
## # A tibble: 200 x 4
##    sample_id compound   concentration   lod
##        <int> <chr>              <dbl> <dbl>
##  1         1 compound_a          18.3  NA  
##  2         1 compound_b         129.   NA  
##  3         2 compound_a          31.3  NA  
##  4         2 compound_b          43.3  NA  
##  5         3 compound_a          26.4  NA  
##  6         3 compound_b          21.4  NA  
##  7         4 compound_a           5    10  
##  8         4 compound_b          10.2  20.5
##  9         5 compound_a          38.0  NA  
## 10         5 compound_b          38.2  NA  
## # … with 190 more rows
```

