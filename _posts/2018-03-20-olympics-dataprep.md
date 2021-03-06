---
title: "Building the Olympics blog: tidy data preparation"
author: "Edwin Thoen"
output: html_document
base-url: https://EdwinTh.github.io
date: "2018-03-20 20:50:00"
tags: [R, data preparation, rvest, tidyverse, web scraping]
---

Last week I did an analysis of [gold diggers at the Olympics](https://edwinth.github.io/analyzing-olympics/). Here I take you along how I scraped the data from wikepedia and subsequently used the tools of the tidyverse to get the data in a format in which they can be analyzed. You are invited to copy-paste the code and see how the data is gradually getting into a shape in which we can analyze it. I will often not show the data in the intermediate steps, because this will clog the blog. Copy the code to find out for yourself.

## Scraping the data

Web scraping in R can be done using the `rvest` package. There are several introductions to this package, such as [this one](https://stat4701.github.io/edav/2015/04/02/rvest_tutorial/). You can use either the Inspector Gadget to find out the html tags of the data you want to scrape, or inspect elements in the web browser. I prefer the latter. These data are published on [Wikipedia](https://en.wikipedia.org/wiki/List_of_2018_Winter_Olympics_medal_winners), for each sport you will find the results in a separate table. It appears that the names of the sports all have `h2` tags, the data of the medal winners all have a `table` class. First we obtain the names of the sports


```r
library(rvest)
library(tidyverse)
library(stringr)
Olympics_2018_wiki <- read_html("https://en.wikipedia.org/wiki/List_of_2018_Winter_Olympics_medal_winners")
sports <- html_nodes(Olympics_2018_wiki, "h2") %>% html_text()
sports %>% head(3)
```

```
## [1] "Alpine skiing[edit]" "Biathlon[edit]"      "Bobsleigh[edit]"
```

Note that you have to use `html_text` to convert the xml nodes to regular R characters. I am not showing the full output here, but it appears that the first fifteen elements contain the sport's names. Also, we still need to get rid of the "[edit]" part of the strings.


```r
sports <- 
  sports %>% 
  `[`(1:15) %>% 
  str_split("\\[") %>% 
  map_chr(1)
```

In the above I call the subsetting operator `[` as a prefix function, rather than its usual usage `object[index]`. (Remember everything in R is a function, even when it appears not be). This to make it pipeable. Every infix operator can be used in this way. If you find this unaesthetic, `magrittr` has alternative functions, such as `extract` to do subsetting. I personally find this a nice use of infixes. Next, we need to get rid of the "[edit]" after each name. We use stringr's `str_split` to split on each "[". This results in a list of character vectors, each vector of length 2. `purrr::map_chr` will select the first element of each vector and store the result in a single character vector.

Now, the tables with the sports


```r
medals <- html_nodes(Olympics_2018_wiki, "table") %>%
  html_table() %>% 
  `[`(3:26)
```

I checked manually that the 3rd up until the 26th table contain the results. Now, if you visit the wiki page you will notice that some sports have a single results table (such as Curling), while others have several (for women's, for men's, and some even for mixed events). I counted on the website the number of tables for each sport.


```r
tables_by_sport <- c(3, 3, 1, 2, 1, 1, 2, 1, 1, 1, 2, 1, 1, 2, 2)
```

The next step is flatten these little data frames to one, and add the sport of each event as a column.


```r
medals_tbl <- 
  medals %>%
  map(~select(.x, Gold, Silver, Bronze)) %>%
  map2_df(rep(sports, tables_by_sport),
          ~mutate(.x, sport = .y)) %>%
  as_data_frame()
```

At the first step we apply dplyr's select to obtain the three columns we are interested in, with `map` this is applied to each of the little data frames. Next we use `map2_df`, this will iterate over two vectors instead of one, to add the sports names to each of the data frames with `mutate`. Note that by using `map2_df`, we bind all the little data frames into one big data frame right away. The result is of class `data.frame`. Since I like to work with tibbles, I coerce it at the final line.


```r
medals_tbl %>% select(Gold) %>% slice(c(4, 16))
```

```
## # A tibble: 2 x 1
##                                                                          Gold
##                                                                         <chr>
## 1                                             "Andr\u00e9 Myhrer\u00a0Sweden"
## 2 "Sweden\u00a0(SWE)Peppe FemlingJesper NelinSebastian SamuelssonFredrik Lind
```

Now we have a little challenge. We want to distill the country names from the strings. However, the names are sometimes at the start of the string and other times at the end. Splitting and selecting the *n*th element, like before, won't work here. To solve this we match every string to every country that won a medal, like so.


```r
medals_tbl %>% select(Gold) %>% slice(c(4, 16)) %>% unlist() %>% 
  str_detect("Sweden")
```

```
## [1] TRUE TRUE
```

First we need to get all the countries that won a medal from the wiki page with the total medal table. The scraping and cleaning is very similar to the ones we already did. 


```r
medal_table_site <- read_html("https://en.wikipedia.org/wiki/2018_Winter_Olympics_medal_table")
medal_table <- medal_table_site %>%
  html_nodes("table") %>%
  `[`(2) %>%
  html_table(fill = TRUE) %>%
  `[[`(1)

countries_with_medal <- medal_table %>%
  pull(NOC) %>%
  `[`(-31) %>% # last element is not a country
  str_sub(end = -7)

# It got an asterix added to the name 
countries_with_medal[7] <- "South Korea"
```

Now, `str_detect` is vectorized over the characters, but it can match to only one pattern at the time. We can wrap over all country names with `map_lgl`. Getting a logical vector that is `TRUE` for the country present, and `FALSE` for all the others. This vector can then subsequently be used to subset the country name.


```r
detect_country <- function(string_with_country, name_vec) {
  ind <- map_lgl(name_vec,
                ~str_detect(string_with_country, pattern = .x))
  countries_with_medal[ind]
}
detect_country("Andr<U+00E9> Myhrer<U+00A0>Sweden", countries_with_medal)
```

```
## [1] "Sweden"
```

This works for a single character, like in the example, in order to get it to work on an entire vector we have to wrap it in `map_chr`.


```r
detect_country_vec <- function(country_vec, name_vec = countries_with_medal) {
  map_chr(country_vec, detect_country, name_vec)
}
```

Now, that is a nice and clean function to extract the country names. However, there are some lines that spoil the party.


```r
medals_tbl %>% slice(23:24)
```

```
## # A tibble: 2 x 4
##                                                     Gold      Silver
##                                                    <chr>       <chr>
## 1       "Canada\u00a0(CAN)Justin KrippsAlexander Kopacz" Not awarded
## 2 "Germany\u00a0(GER)Francesco FriedrichThorsten Margis" Not awarded
## # ... with 2 more variables: Bronze <chr>, sport <chr>
```

We have a shared Gold in the bobsleigh, spread over two lines. For the Gold itself it doesn't cause a problem, but for the Silver the function breaks, and for the Bronze Latvia would be counted twice for one medal.


```r
medals_tbl %>% slice(62)
```

```
## # A tibble: 1 x 4
##                                         Gold
##                                        <chr>
## 1 "Tobias Wendl\nand Tobias Arlt\u00a0(GER)"
## # ... with 3 more variables: Silver <chr>, Bronze <chr>, sport <chr>
```

Another spoiler. For some reason in the luge abbreviations are used instead of the full names. Pfff, we have no match here. There are several more exceptions that make our function break.

During this type of analyses you are almost always confronted with the choice between manual labor and automation (writing a general purpose function) several times. I use the following heuristics for this choice:

1) How much more work takes automation compared to manual labor? If little, automate.

2) Is the code likely to be run on data other than the current? If yes, probably automate.

3) Is a general function portable to and useful in other projects? If yes, most definitively automate.

In this case, should we incorporate the exceptions in the function, or do we just do them by hand? It is a lot more work to automate because of the many different exceptions. No, we are not expecting new data to flow through this. And finally, these problems seem very specific for this problem, a general purpose function is not likely to make our future life easier. Manual labor it is. By trial and error we find the problem lines, discard them, apply the function and add the countries for the problem lines manually.


```r
problem_lines <- c(23:26, 34, 41, 45, 46, 62)
happy_flow <- medals_tbl[-problem_lines, ] %>%
  mutate_at(.vars = vars(Gold, Silver, Bronze), .funs = detect_country_vec)
hand_work <- medals_tbl[problem_lines, ] %>%
  mutate(Gold = c("Canada", "Germany", "Germany", NA, "Norway", "Sweden", "Germany", "Canada", "Germany"),
         Silver = c(NA, NA, "South Korea", "Germany", "Sweden", "South Korea", "China", "France", "Austria"),
         Bronze = c("Latvia", NA, NA, NA, "Norway", "Japan", "Canada", "United States", "Germany")) %>%
  bind_rows(data_frame(Gold = NA, Silver = NA, Bronze = "Finland", sport = "Cross-country skiing"))

medal_set <- bind_rows(happy_flow, hand_work) %>%
  gather(medal, country, -sport) %>%
  filter(!is.na(country))
```

Note the use of the nice `mutate_at` function. In one single line we replace the original strings by the values extracted by our function for all three columns. In the final lines `gather` (tidyr package) is applied to transform the data from wide to long. Each row is now a medal won.


```r
medal_set %>% head(2)
```

```
## # A tibble: 2 x 3
##           sport medal country
##           <chr> <chr>   <chr>
## 1 Alpine skiing  Gold  Norway
## 2 Alpine skiing  Gold Austria
```

Finally, since we are interested in the number of medals per sport per country. We can already aggregated. 


```r
medal_set_sum <- medal_set %>% 
  count(sport, medal, country) %>% 
  rename(medal_count = n)

medal_set_sum %>% head()
```

```
## # A tibble: 6 x 4
##           sport  medal       country medal_count
##           <chr>  <chr>         <chr>       <int>
## 1 Alpine skiing Bronze       Austria           2
## 2 Alpine skiing Bronze        France           2
## 3 Alpine skiing Bronze         Italy           1
## 4 Alpine skiing Bronze Liechtenstein           1
## 5 Alpine skiing Bronze        Norway           2
## 6 Alpine skiing Bronze   Switzerland           2
```

I showed you how the great tidyverse toolbox can be used to get raw data from the web, and transform it into a clean set that is ready for analysis.

## Some remarks on web scraping

A disadvantage of using web data as a source, is that the layout of the data might change. My pipeline broke several times, because changes were made to the wiki. Because of this, it is not assured this code will run in future times. For this example I kept the pipeline live, because I wanted to do this blog post including the scraping. However, it would have saved me a good deal of trouble if I stored the set in a csv the first time I had a proper version of the data.
