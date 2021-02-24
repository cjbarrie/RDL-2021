---
title: "Researching Digital Life"
subtitle: "Lecture 7: Screen-scraping and APIs"
author:
  name: Christopher Barrie
  affiliation: University of Edinburgh | [RDL](https://github.com/cjbarrie/RDL-Ed)
# date: Lecture 6  #"24 February 2021"
output: 
  html_document:
    theme: flatly
    highlight: haddock
    # code_folding: show
    toc: yes
    toc_depth: 4
    toc_float: yes
    keep_md: true
    
bibliography: RDL.bib    
---


# Week 7: Screen-scraping and APIs

This week you had readings by @Freelon2018a, @Bruns2019, @Puschmann2019, and @Lazer2020b as well as a [report](https://www.disinfobservatory.org/download/26541) by SOMA outlining solutions for research data exchange. 

The hands-on exercise for this week uses different sources of online data, and here I introduce you to how we might gather data through both screen-scraping (or server-side) techniques as well as API (or client-side) techniques.

## Week 7 Exercise 

In this tutorial, you will learn how to summarise, aggregate, and analyze text in R:

* How to select elements of CSS using SelectorGadget
* How to use the <tt>rvest</tt> package to scrape CSS elements
* How to use the Twitter API

## Setup 

To practice these skills, we will use a series of webpages on the Internet Archive that host material collected at the Arab Spring protests in Egypt in 2011. The original website can be seen [here](https://www.tahrirdocuments.org/) and below.

<iframe src="https://www.tahrirdocuments.org/" width="100%" height="400px"></iframe>

This might sound complicated but it isn't really. In essence, APIs simply provide data in a more usable format without the need for alternative techniques such as web scraping. Be warned, too, that some websites do not permit automated web scraping, meaning the use of an API is essential.

##  Load data and packages 

Beforce proceeding, we'll load the remaining packages we will need for this tutorial.


```r
library(tidyverse) # loads dplyr, ggplot2, and others
library(ggthemes) # includes a set of themes to make your visualizations look nice!
library(readr) # more informative and easy way to import data
library(stringr) # to handle text elements
library(rvest) #for scraping
```

We can download the final dataset we will produce with:


```r
pamphdata <- read_csv("data/pamphlets_formatted_gsheets.csv")
```

```
## 
## ── Column specification ────────────────────────────────────────────────────────
## cols(
##   title = col_character(),
##   date = col_date(format = ""),
##   year = col_double(),
##   text = col_character(),
##   tags = col_character(),
##   imageurl = col_character(),
##   imgID = col_character(),
##   image = col_character()
## )
```

If you're working on this document from your own computer ("locally") you can download the Edinburgh Fringe data in the following way:


```r
pamphdata <- read_csv("https://raw.githubusercontent.com/cjbarrie/RDL-Ed/main/03-screenscrape-apis/data/pamphlets_formatted_gsheets.csv")
```


## Inspect and filter data 

Let's have a look at what we will end up producing:


```r
colnames(pamphdata)
```

```
## [1] "title"    "date"     "year"     "text"     "tags"     "imageurl" "imgID"   
## [8] "image"
```

And then: 


```r
glimpse(pamphdata)
```

```
## Rows: 523
## Columns: 8
## $ title    <chr> "The Season of Anger Sets in Among the Arab Peoples", "The M…
## $ date     <date> 2011-03-30, 2011-03-30, 2011-03-30, 2011-03-30, 2011-03-30,…
## $ year     <dbl> 2011, 2011, 2011, 2011, 2011, 2011, 2011, 2011, 2011, 2011, …
## $ text     <chr> "The Season of Anger Sets in Among the Arab Peoples,,A membe…
## $ tags     <chr> "Solidarity", "Solidarity, Workers", "Solidarity, Workers", …
## $ imageurl <chr> "https://wayback.archive-it.org/2358/20120130161341im_/http:…
## $ imgID    <chr> "imgID1", "imgID2", "imgID3", "imgID4", "imgID5", "imgID6", …
## $ image    <chr> "=Arrayformula(image(F2,2))", NA, NA, NA, NA, NA, NA, NA, NA…
```

## Scraping webpage content

We are going to return to the Internet Archived webpages to see how we can produce this final formatted dataset. The archived Tahrir Documents webpages can be accessed [here](https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/).

We first need to inspect the URL structures of the documents we want to capture. When we scroll down the page we see listed a number of documents. Each of these directs to an individual pamphlet distributed at protests during the 2011 Egyptian Revolution. 

When we scroll to the very bottom of the page, we see listed a number of hyperlinks to documents stored by month.


![alt text here](images/tahrir_archives.png)

Click on one of these and see how the URL changes.

We see that if our starting URL was:




```
https://wayback.archive-it.org/2358/20120130135111/http://www.tahrirdocuments.org/
```

Then if we click on March 2011, the first month for which we have documents, we see that the url becomes:




```
https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/2011/03/
```

, for August 2011 it becomes:


```
https://wayback.archive-it.org/2358/20120130142155/http://www.tahrirdocuments.org/2011/08/
```

, and for January 2012 it becomes:


```
https://wayback.archive-it.org/2358/20120130142014/http://www.tahrirdocuments.org/2012/01/
```

We notice that for each month, the URL changes with the addition of month and year between back slashes at the end or the URL. 

We are going to want to retrieve the text of documents archived for each month. As such, our first task is to store each of these webpages as a series of strings. We could do this manually by, for example, pasting year and month strings to the end of each URL for each month from March, 2011 to January, 2012:


```r
url <- "https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/"

url1 <- paste0(url,"2011/03/")
url2 <- paste0(url,"2011/04/")
url3 <- paste0(url,"2011/04/")

#etc...

urls <- c(url1, url2, url3)
```

But this wouldn't be particularly efficient...

Instead, we can wrap all of this in a loop. 


```r
urls <- character(0)
for (i in 3:13) {
  url <- "https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/"
  newurl <- ifelse(i <10, paste0(url,"2011/0",i,"/"), 
                   ifelse(i>=10 & i<=12 , paste0(url,"2011/",i,"/"), 
                          paste0(url,"2012/01/")))
  urls <- c(urls, newurl)
}
```

Here, we are first specifying the starting URL as above. We are then iterating through the numbers 3 to 13. And we are telling R to take the new URL and then, depending on the number in the loop we are on, to take the base starting url--- https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/ --- and to paste on the end of it the string "2011/0", then the number of the loop we are on, and then "/". So, for the first "i" in the loop---the number 3---then we are effectively calling the equivalent of:


```r
i
```

```
## [1] 13
```

```r
url <- "https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/"
newurl <- paste0(url,"2011/0",3,"/")
```

Which gives:


```
## [1] "https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/2011/03/"
```

In the above, the `ifelse()` commands are simply telling R: if i (the number of the loop we are on) is less than 10 then `paste0(url,"2011/0",i,"/")`; i.e., if i is less than 10 then paste "2011/0", then "i" and then "/". So for the number 3 this becomes:

`"https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/2011/03/"` 

, and for the number 4 this becomes 

`"https://wayback.archive-it.org/2358/20120130143023/http://www.tahrirdocuments.org/2011/04/"` 

## References 