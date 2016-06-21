Spark Interface for R
================

A set of tools to provision, connect and interface to Apache Spark from within the R language and ecosystem. This package supports connecting to local and remote Apache Spark clusters and provides support for R packages like dplyr and DBI.

Installation
------------

You can install the development version of the **sparklyr** package using **devtools** as follows (note that installation of the development version of **devtools** itself is also required):

``` r
devtools::install_github("hadley/devtools")
devtools::reload(devtools::inst("devtools"))

devtools::install_github("rstudio/sparklyr", auth_token = "1296316f10e7fe4adc675c77366265b5f180933d")
```

You can then install various versions of Spark using the `spark_install` function:

``` r
library(sparklyr)
spark_install(version = "1.6.1", hadoop_version = "2.6", reset = TRUE)
```

dplyr Interface
---------------

The sparklyr package implements a dplyr back-end for Spark. Connect to Spark using the `spark_connect` function then use the returned connection as a remote dplyr source.

``` r
# connect to local spark instance 
library(sparklyr)
sc <- spark_connect("local", version = "1.6.1")
```

Now we copy some datasets from R into the Spark cluster:

``` r
iris_tbl <- copy_to(sc, iris)
flights_tbl <- copy_to(sc, flights)
batting_tbl <- copy_to(sc, Batting, "batting")
src_tbls(sc)
```

    ## [1] "batting" "flights" "iris"

Then you can run dplyr against Spark:

``` r
# filter by departure delay and print the first few records
flights_tbl %>% filter(dep_delay == 2)
```

    ## Source:   query [?? x 16]
    ## Database: spark connection master=local app=sparklyr local=TRUE
    ## 
    ## # S3: tbl_spark
    ##     year month   day dep_time dep_delay arr_time arr_delay carrier tailnum
    ##    <int> <int> <int>    <int>     <dbl>    <int>     <dbl>   <chr>   <chr>
    ## 1   2013     1     1      517         2      830        11      UA  N14228
    ## 2   2013     1     1      542         2      923        33      AA  N619AA
    ## 3   2013     1     1      702         2     1058        44      B6  N779JB
    ## 4   2013     1     1      715         2      911        21      UA  N841UA
    ## 5   2013     1     1      752         2     1025        -4      UA  N511UA
    ## 6   2013     1     1      917         2     1206        -5      B6  N568JB
    ## 7   2013     1     1      932         2     1219        -6      VX  N641VA
    ## 8   2013     1     1     1028         2     1350        11      UA  N76508
    ## 9   2013     1     1     1042         2     1325        -1      B6  N529JB
    ## 10  2013     1     1     1231         2     1523        -6      UA  N402UA
    ## ... with more rows, and 7 more variables: flight <int>, origin <chr>,
    ##   dest <chr>, air_time <dbl>, distance <dbl>, hour <dbl>, minute <dbl>

[Introduction to dplyr](https://cran.rstudio.com/web/packages/dplyr/vignettes/introduction.html) provides additional dplyr examples you can try. For example, consider the last example from the tutorial which plots data on flight delays:

``` r
delay <- flights_tbl %>% 
  group_by(tailnum) %>%
  summarise(count = n(), dist = mean(distance), delay = mean(arr_delay)) %>%
  filter(count > 20, dist < 2000, !is.na(delay)) %>%
  collect

# plot delays
library(ggplot2)
ggplot(delay, aes(dist, delay)) +
  geom_point(aes(size = count), alpha = 1/2) +
  geom_smooth() +
  scale_size_area(max_size = 2)
```

![](res/ggplot2-1.png)

### Window Functions

dplyr [window functions](https://cran.r-project.org/web/packages/dplyr/vignettes/window-functions.html) are also supported, for example:

``` r
batting_tbl %>%
  select(playerID, yearID, teamID, G, AB:H) %>%
  arrange(playerID, yearID, teamID) %>%
  group_by(playerID) %>%
  filter(min_rank(desc(H)) <= 2 & H > 0)
```

    ## Source:   query [?? x 7]
    ## Database: spark connection master=local app=sparklyr local=TRUE
    ## Groups: playerID
    ## 
    ## # S3: tbl_spark
    ##     playerID yearID teamID     G    AB     R     H
    ##        <chr>  <int>  <chr> <int> <int> <int> <int>
    ## 1  anderal01   1941    PIT    70   223    32    48
    ## 2  anderal01   1942    PIT    54   166    24    45
    ## 3  balesco01   2008    WAS    15    15     1     3
    ## 4  balesco01   2009    WAS     7     8     0     1
    ## 5  bandoch01   1986    CLE    92   254    28    68
    ## 6  bandoch01   1984    CLE    75   220    38    64
    ## 7  bedelho01   1962    ML1    58   138    15    27
    ## 8  bedelho01   1968    PHI     9     7     0     1
    ## 9  biittla01   1977    CHN   138   493    74   147
    ## 10 biittla01   1975    MON   121   346    34   109
    ## ... with more rows

ML Functions
------------

MLlib functions are also supported, see [ml samples](docs/ml_examples.md). For instasnce, k-means can be run as:

``` r
model <- iris_tbl %>%
  select(Petal_Width, Petal_Length) %>%
  ml_kmeans(centers = 3)

iris_tbl %>%
  select(Petal_Width, Petal_Length) %>%
  collect %>%
  ggplot(aes(Petal_Length, Petal_Width)) +
    geom_point(data = model$centers, aes(Petal_Width, Petal_Length), size = 60, alpha = 0.1) +
    geom_point(aes(Petal_Width, Petal_Length), size = 2, alpha = 0.5)
```

![](res/unnamed-chunk-6-1.png)

Reading and Writing Data
------------------------

``` r
  temp_csv <- tempfile(fileext = ".csv")
  temp_parquet <- tempfile(fileext = ".parquet")
  temp_json <- tempfile(fileext = ".json")
  
  spark_write_csv(iris_tbl, temp_csv)
  iris_csv_tbl <- spark_read_csv(sc, "iris_csv", temp_csv)
  
  spark_write_parquet(iris_tbl, temp_parquet)
  iris_parquet_tbl <- spark_read_parquet(sc, "iris_parquet", temp_parquet)
  
  spark_write_csv(iris_tbl, temp_json)
  iris_json_tbl <- spark_read_csv(sc, "iris_json", temp_json)
  
  src_tbls(sc)
```

    ## [1] "batting"      "flights"      "iris"         "iris_csv"    
    ## [5] "iris_json"    "iris_parquet"

Extensibility
-------------

Spark provides low level access to native JVM objects, this topic targets users creating packages based on low-level spark integration. Here's an example of an R `count_lines` function built by calling Spark functions for reading and counting the lines of a text file.

``` r
library(sparkapi)

# define an R interface to Spark line counting
count_lines <- function(sc, path) {
  sparkapi_spark_context(sc) %>% 
    sparkapi_invoke("textFile", path, as.integer(1)) %>% 
      sparkapi_invoke("count")
}

# write a CSV 
tempfile <- tempfile(fileext = ".csv")
write.csv(nycflights13::flights, tempfile, row.names = FALSE, na = "")

# call spark to count the lines
count_lines(sc, tempfile)
```

    ## [1] 336777

Package authors can use this mechanism to create an R interface to any of Spark's underlying Java APIs.

dplyr Utilities
---------------

You can cache a table into memory with:

``` r
tbl_cache(sc, "batting")
```

and unload from memory using:

``` r
tbl_uncache(sc, "batting")
```

Connection Utilities
--------------------

You can view the Spark web console using the `spark_web` function:

``` r
spark_web(sc)
```

You can show the log using the `spark_log` function:

``` r
spark_log(sc, n = 10)
```

    ## 16/06/21 10:23:53 INFO BlockManagerInfo: Removed broadcast_90_piece0 on localhost:58376 in memory (size: 4.6 KB, free: 486.9 MB)
    ## 16/06/21 10:23:53 INFO ContextCleaner: Cleaned accumulator 198
    ## 16/06/21 10:23:53 INFO BlockManagerInfo: Removed broadcast_89_piece0 on localhost:58376 in memory (size: 8.5 KB, free: 486.9 MB)
    ## 16/06/21 10:23:53 INFO ContextCleaner: Cleaned accumulator 197
    ## 16/06/21 10:23:53 INFO ContextCleaner: Cleaned shuffle 20
    ## 16/06/21 10:23:53 INFO Executor: Finished task 0.0 in stage 67.0 (TID 465). 2082 bytes result sent to driver
    ## 16/06/21 10:23:53 INFO TaskSetManager: Finished task 0.0 in stage 67.0 (TID 465) in 95 ms on localhost (1/1)
    ## 16/06/21 10:23:53 INFO TaskSchedulerImpl: Removed TaskSet 67.0, whose tasks have all completed, from pool 
    ## 16/06/21 10:23:53 INFO DAGScheduler: ResultStage 67 (count at NativeMethodAccessorImpl.java:-2) finished in 0.095 s
    ## 16/06/21 10:23:53 INFO DAGScheduler: Job 46 finished: count at NativeMethodAccessorImpl.java:-2, took 0.097874 s

Finally, we disconnect from Spark:

``` r
spark_disconnect(sc)
```

Additional Resources
--------------------

For performance runs under various parameters, read: [Dplyr Performance](docs/perf_dplyr.md) and [Spark 1B-Rows Performance](docs/perf_1b.md)
