Data Processing Document
================

  - [Tidy data and join additional
    information](#tidy-data-and-join-additional-information)
  - [Filter on drivers that drove for multiple constructors (not
    used)](#filter-on-drivers-that-drove-for-multiple-constructors-not-used)
  - [EDA on Laptimes](#eda-on-laptimes)
  - [Compute Average Lap Times](#compute-average-lap-times)
  - [Save final processed dataset](#save-final-processed-dataset)

## Tidy data and join additional information

``` r
# throw out extraneous/irrelevant columns
df_results_trim <- 
  df_results %>%
  select(-c(time, position, positionText, points, grid, number)) %>%
  select(-c(milliseconds, rank, fastestLap, fastestLapTime))

# select which attributes to keep about each driver
df_drivers_trim <-
  df_drivers %>%
  unite(driver_name, c(forename, surname), sep = " ") %>% # driver's full name
  select(driverId, driver_name) #, driver_nationality=nationality)
df_results_plusdrivers <- left_join(df_results_trim, df_drivers_trim, by = "driverId")

# select which attributes to keep about each constructor
df_constructors_trim <-
  df_constructors %>%
  select(constructorId, constructor_name=name) #, constructor_nationality=nationality)
df_results_plusconstructors <- left_join(df_results_plusdrivers, df_constructors_trim, by = "constructorId")

# select which attributes to keep about each race
df_circuits_trim <-
  df_races %>%
  select(raceId, year, round, circuitId, race_name=name)
df_results_plusraces <- left_join(df_results_plusconstructors, df_circuits_trim, by = "raceId")

# get status from statusId
df_results_plusstatus <- left_join(df_results_plusraces, df_status, by = "statusId")

# select which attributes to keep about each circuit
df_circuits_trim <-
  df_circuits %>% 
  select(circuitId, circuit_name=name) #, circuit_country=country) 
df_results_pluscircuits <- left_join(df_results_plusstatus, df_circuits_trim, by = c("circuitId"))

# turn all \\N into NAs
df_clean <-
  df_results_pluscircuits %>%
  mutate_all(na_if, "\\N")
df_clean
```

    ## # A tibble: 24,900 x 16
    ##    resultId raceId driverId constructorId positionOrder  laps fastestLapSpeed
    ##       <dbl>  <dbl>    <dbl>         <dbl>         <dbl> <dbl> <chr>          
    ##  1        1     18        1             1             1    58 218.300        
    ##  2        2     18        2             2             2    58 217.586        
    ##  3        3     18        3             3             3    58 216.719        
    ##  4        4     18        4             4             4    58 215.464        
    ##  5        5     18        5             1             5    58 218.385        
    ##  6        6     18        6             3             6    57 212.974        
    ##  7        7     18        7             5             7    55 213.224        
    ##  8        8     18        8             6             8    53 217.180        
    ##  9        9     18        9             2             9    47 215.100        
    ## 10       10     18       10             7            10    43 213.166        
    ## # ... with 24,890 more rows, and 9 more variables: statusId <dbl>,
    ## #   driver_name <chr>, constructor_name <chr>, year <dbl>, round <dbl>,
    ## #   circuitId <dbl>, race_name <chr>, status <chr>, circuit_name <chr>

``` r
write.csv(df_clean,"processed_data/clean_f1.csv", row.names = FALSE)
```

## Filter on drivers that drove for multiple constructors (not used)

``` r
df_multi_drivers <-
  df_clean %>%
  
  # get one datapoint of each pair of driver and constructor
  group_by(driver_name, constructor_name) %>%
  filter(row_number() == 1) %>%
  
  # get how many constructors that each driver has driven for
  group_by(driver_name) %>%
  mutate(driver_numconstr = n()) %>%
  
  # keep drivers that drove for more than one constructor
  filter(driver_numconstr > 1, row_number() == 1) %>%
  select(driver_name) %>% 
  arrange(driver_name)
df_multi_drivers
```

    ## # A tibble: 499 x 1
    ## # Groups:   driver_name [499]
    ##    driver_name         
    ##    <chr>               
    ##  1 Adrian Sutil        
    ##  2 Aguri Suzuki        
    ##  3 Al Herman           
    ##  4 Al Keller           
    ##  5 Alain Prost         
    ##  6 Alan Jones          
    ##  7 Alan Rees           
    ##  8 Alberto Ascari      
    ##  9 Alberto Colombo     
    ## 10 Alessandro de Tomaso
    ## # ... with 489 more rows

``` r
# get all of the races "multi drivers" drove
df_results_multi_drivers <- inner_join(df_clean, df_multi_drivers, by = "driver_name")
df_results_multi_drivers %>% 
  arrange(driver_name)
```

    ## # A tibble: 23,059 x 16
    ##    resultId raceId driverId constructorId positionOrder  laps fastestLapSpeed
    ##       <dbl>  <dbl>    <dbl>         <dbl>         <dbl> <dbl> <chr>          
    ##  1       16     18       16            10            16     8 207.461        
    ##  2       42     19       16            10            20     5 198.891        
    ##  3       63     20       16            10            19    56 204.136        
    ##  4       87     21       16            10            21     0 <NA>           
    ##  5      104     22       16            10            16    57 216.454        
    ##  6      123     23       16            10            15    67 146.564        
    ##  7      148     24       16            10            20    13 194.624        
    ##  8      167     25       16            10            19    69 202.385        
    ##  9      186     26       16            10            18    10 188.545        
    ## 10      203     27       16            10            15    67 211.408        
    ## # ... with 23,049 more rows, and 9 more variables: statusId <dbl>,
    ## #   driver_name <chr>, constructor_name <chr>, year <dbl>, round <dbl>,
    ## #   circuitId <dbl>, race_name <chr>, status <chr>, circuit_name <chr>

## EDA on Laptimes

``` r
pits <- df_pitstops %>% 
  filter(driverId == 1, raceId == 841) %>% 
  pull(lap)

df_laptimes %>% 
  filter(driverId == 1, raceId == 841) %>% 
  ggplot(aes(lap, milliseconds  / 1000 / 60)) + 
  geom_vline(xintercept = pits, linetype = 2, color = "grey") +
  geom_line() +
  geom_point(color = "blue") + 
  labs(
    x = "Lap Number",
    y = "Lap Time (Minutes)"
  )
```

![](dataProcessingDocument_files/figure-gfm/lap%20and%20laptimes%20visualization%20for%20specific%20race%20and%20racer-1.png)<!-- -->

``` r
df_laptimes %>% 
  filter(raceId == 841) %>% 
  mutate(driverId = as.factor(driverId))%>% 
  ggplot(aes(lap, milliseconds / 1000 / 60)) +
  geom_line(aes(color = driverId)) + 
  ylim(1.5, 2.5) + 
  labs(
    x = "Lap Number",
    y = "Lap Time (Minutes)"
  )
```

    ## Warning: Removed 1 row(s) containing missing values (geom_path).

![](dataProcessingDocument_files/figure-gfm/lap%20number%20&%20laptimes%20for%20all%20racers%20in%20a%20given%20race-1.png)<!-- -->

## Compute Average Lap Times

``` r
df_avglaptime <-
  df_laptimes %>% 
  group_by(driverId, raceId) %>%
  summarize(total_time = sum(milliseconds), avg_lap = total_time / n())
```

    ## `summarise()` regrouping output by 'driverId' (override with `.groups` argument)

``` r
df_avglaptime
```

    ## # A tibble: 9,233 x 4
    ## # Groups:   driverId [130]
    ##    driverId raceId total_time avg_lap
    ##       <dbl>  <dbl>      <dbl>   <dbl>
    ##  1        1      1    5658698  97564.
    ##  2        1      2    3391355 109399.
    ##  3        1      3    7135351 127417.
    ##  4        1      4    5530278  97022.
    ##  5        1      5    5840216  89849.
    ##  6        1      6    6086882  79050.
    ##  7        1      7    5265302  90781.
    ##  8        1      8    5032322  85294.
    ##  9        1      9    5887232  99784.
    ## 10        1     10    5903876  84341.
    ## # ... with 9,223 more rows

``` r
df_with_avglaptime <- left_join(df_clean, df_avglaptime, by = c("driverId", "raceId"))
df_with_avglaptime <-
  df_with_avglaptime %>% 
  filter(!is.na(avg_lap))

df_with_avglaptime <-
  df_with_avglaptime %>% 
  group_by(circuitId) %>%
  mutate(circuit_avg_lap = mean(avg_lap), circuit_lap_sd = sd(avg_lap)) %>%
  mutate(std_avg_lap = (avg_lap-circuit_avg_lap)/circuit_lap_sd)
df_with_avglaptime
```

    ## # A tibble: 9,233 x 21
    ## # Groups:   circuitId [37]
    ##    resultId raceId driverId constructorId positionOrder  laps fastestLapSpeed
    ##       <dbl>  <dbl>    <dbl>         <dbl>         <dbl> <dbl> <chr>          
    ##  1        1     18        1             1             1    58 218.300        
    ##  2        2     18        2             2             2    58 217.586        
    ##  3        3     18        3             3             3    58 216.719        
    ##  4        4     18        4             4             4    58 215.464        
    ##  5        5     18        5             1             5    58 218.385        
    ##  6        6     18        6             3             6    57 212.974        
    ##  7        7     18        7             5             7    55 213.224        
    ##  8        8     18        8             6             8    53 217.180        
    ##  9        9     18        9             2             9    47 215.100        
    ## 10       10     18       10             7            10    43 213.166        
    ## # ... with 9,223 more rows, and 14 more variables: statusId <dbl>,
    ## #   driver_name <chr>, constructor_name <chr>, year <dbl>, round <dbl>,
    ## #   circuitId <dbl>, race_name <chr>, status <chr>, circuit_name <chr>,
    ## #   total_time <dbl>, avg_lap <dbl>, circuit_avg_lap <dbl>,
    ## #   circuit_lap_sd <dbl>, std_avg_lap <dbl>

## Save final processed dataset

``` r
write.csv(df_with_avglaptime,"processed_data/avglaptime.csv", row.names = FALSE)
```

``` r
write.csv(df_with_avglaptime,"processed_data/std_avg_laptime.csv", row.names = FALSE)
```
