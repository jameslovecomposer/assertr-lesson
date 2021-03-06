# assertr-lesson

*Note: Adapted from Andrew MacDonald's teaching materials for UBC BIOL548. Thanks Andrew!*

## Introduction

You've recently obtained some data. How can you tell if it is correct? What if you have:

* Many datasets
* Data collected by many field assistants
* Data collected by you when you were sleep deprived
* Data produced by many collaborators
* Data from a past project in the lab 
* meta-analysis datasets
* Internet data, from an API or from scraping
* data from a particularly _experimental_ simulation?

How can you tell if the numbers you have make any sense? (also called "sanity checking")

We'll try out the practice of asserting data by using the gapminder dataset:

```r
library(gapminder)
```

And we'll be using an R package specifically designed to work with data assertions: `assertr`, by Tony Fischetti ([github page](https://github.com/tonyfischetti/assertr) and [rOpenSci community call with Tony](https://vimeo.com/141906295))


```r
# install.packages("assertr")
library(assertr)
library(dplyr)
```

## What do we know about the dataset

The first step in checking whether we have a problem in a dataset is to try to think of what reasonable properties of the data *should* be.

Let's begin by looking at the top of gapminder:


```r
knitr::kable(head(gapminder))
```



country       continent    year   lifeExp        pop   gdpPercap
------------  ----------  -----  --------  ---------  ----------
Afghanistan   Asia         1952    28.801    8425333    779.4453
Afghanistan   Asia         1957    30.332    9240934    820.8530
Afghanistan   Asia         1962    31.997   10267083    853.1007
Afghanistan   Asia         1967    34.020   11537966    836.1971
Afghanistan   Asia         1972    36.088   13079460    739.9811
Afghanistan   Asia         1977    38.438   14880372    786.1134

* `country`: should probably be a country that exists in the world. Any additions to gapminder should contain the same countries, spelt the same way.
* `continent`: similarly, there are only five continents (`unique(gapminder$continent)`) and any new data should match those.
* `lifeExp`: must always be a positive real number
* `pop`: always a positive integer
* `gdpPercap`: 

## Data checking in base R

What base R functions exist for checking the same thing?


```r
stopifnot(1 == 1, all.equal(pi, 3.14159265), 1 < 2) # all TRUE

stopifnot(gapminder$lifeExp > 0)
```

perfectly good way to test data! However, not perfectly flexible, and difficult to chain:


```r
# gapminder %>% 
#   {stopifnot(.[["lifeExp"]] > 0);stopifnot(.[["pop"]] < 0)}
```
.. that's rather awkward.

Fortunately, we have `assertr`!

Assertr gives us several functions. They take the dataset as the first argument (perfect for piping) and run a test on it. If it fails, the function causes an error. If everything is OK, they  return the dataset so that it can be piped to a new test (or into an analysis)

### `verify`

The most straightforward function in assertr is `verify()`. It evaluates a logical expression (`>` or `==` or `<` etc) using a data frame. That lets us check some general properties of a dataset. 


```r
gm <- gapminder %>% 
  verify(nrow(.) == 1704) %>% 
  verify(ncol(.) == 6) %>% 
  verify(is.factor(.$continent)) %>% 
  verify(length(levels(.$continent)) == 5)
```

So far so good! While this is good for checking the dataset at a coarse level, it doesn't tell us *where* the unusual numbers are:


```r
# gapminder %>% 
#   verify(lifeExp > 30)
```

### `assert`

`assert` evaluates a _predicate function_ on a column of a dataset, and identifies where the predicate returns `FALSE`.

* a predicate function is any function that will give you `TRUE` or `FALSE` for every element in a vector. `is.na()` is a common example.

Here, we can't write `pop > 0`, we need to write this in terms of a function. Fortunately, `assertr` supplies some flexible predicates for us:

```r
gm2 <- gapminder %>% 
  assert(within_bounds(0,Inf), pop)
```

### `assert` has the power of `dplyr::select`

You can use the same syntax from `dplyr` to select columns in `assertr`. That means less typing!


```r
gm2 <- gapminder %>% 
  assert(within_bounds(0, Inf), lifeExp:gdpPercap)
```

Let's meet the other handy predicates:


```r
## let's create a vector of all continents
all_continents <- levels(gapminder$continent)
all_continents
```

```
## [1] "Africa"   "Americas" "Asia"     "Europe"   "Oceania"
```

```r
gm2 <- gapminder %>% 
  ## check for missing values
  assert(not_na, country:gdpPercap) %>% 
  ## check that all continents are matching
  assert(in_set(all_continents), continent) %>% 
  assert(within_bounds(0, Inf), lifeExp:gdpPercap)
```

Let's create a new version of gapminder that has some data entry mistakes

```r
gapminder_mistake <- gapminder 

levels(gapminder_mistake$continent)[levels(gapminder_mistake$continent) == "Africa"] <- "Afrrrrica"
```

Let's see if we can detect the spelling mistake

```r
gapminder_mistake %>% 
  ## check for missing values
  assert(not_na, country:gdpPercap) %>% 
  ## check that all continents are matching
  assert(in_set(all_continents), continent) %>% 
  assert(within_bounds(0, Inf), lifeExp:gdpPercap)
```

We can also cook up our own predicate: 


```r
is_factor <- function(x) is.factor(x)

gm_fac <- gapminder %>% 
  assert(is_factor, country, continent)
```

>  *challenge!* 
write a function to test if population is an integer. (Note that it is probably not stored as an integer in your present data.frame)

### Some notes on the `assertr` functions (from the [assertr github page](https://github.com/ropenscilabs/assertr))

- `verify` - takes a data frame (its first argument is provided by
the `%>%` operator above), and a logical (boolean) expression. Then, `verify`
evaluates that expression using the scope of the provided data frame. If any
of the logical values of the expression's result are `FALSE`, `verify` will
raise an error that terminates any further processing of the pipeline.

- `assert` - takes a data frame, a predicate function, and an arbitrary
number of columns to apply the predicate function to. The predicate function
(a function that returns a logical/boolean value) is then applied to every
element of the columns selected, and will raise an error if it finds any
violations.

- `insist` - takes a data frame, a predicate-generating function, and an
arbitrary number of columns. For each column, the the predicate-generating
function is applied, returning a predicate. The predicate is then applied to
every element of the columns selected, and will raise an error if it finds any
violations. The reason for using a predicate-generating function to return a
predicate to use against each value in each of the selected rows is so
that, for example, bounds can be dynamically generated based on what the data
look like; this the only way to, say, create bounds that check if each datum is
within x z-scores, since the standard deviation isn't known a priori.
Internally, the `insist` function uses `dplyr`'s `select` function to extract
the columns to test the predicate function on.

- `assert_rows` - takes a data frame, a row reduction function, a predicate
function, and an arbitrary number of columns to apply the predicate function
to. The row reduction function is applied to the data frame, and returns a value
for each row. The predicate function is then applied to every element of vector
returned from the row reduction function, and will raise an error if it finds
any violations. This functionality is useful, for example, in conjunction with
the `num_row_NAs()` function to ensure that there is below a certain number of
missing values in each row. Internally, the `assert_rows` function uses
`dplyr`'s`select` function to extract the columns to test the predicate
function on.

- `insist_rows` - takes a data frame, a row reduction function, a
predicate-generating
function, and an arbitrary number of columns to apply the predicate function
to. The row reduction function is applied to the data frame, and returns a value
for each row. The predicate-generating function is then applied to the vector
returned from the row reduction function and the resultant predicate is
applied to each element of that vector. It will raise an error if it finds any
violations. This functionality is useful, for example, in conjunction with
the `maha_dist()` function to ensure that there are no flagrant outliers.
Internally, the `assert_rows` function uses `dplyr`'s`select` function to
extract the columns to test the predicate function on.


`assertr` also offers three (so far) predicate functions designed to be used
with the `assert` and `assert_rows` functions:

- `not_na` - that checks if an element is not NA
- `within_bounds` - that returns a predicate function that checks if a numeric
value falls within the bounds supplied, and
- `in_set` - that returns a predicate function that checks if an element is
a member of the set supplied.

and predicate generators designed to be used with the `insist` and `insist_rows`
functions:

- `within_n_sds` - used to dynamically create bounds to check vector elements with
based on standard z-scores
- `within_n_mads` - better method for dynamically creating bounds to check vector
elements with based on 'robust' z-scores (using median absolute deviation)

and the following row reduction functions designed to be used with `assert_rows`
and `insist_rows`:

- `num_row_NAs` - counts number of missing values in each row
- `maha_dist` - computes the mahalanobis distance of each row (for outlier
detection). It will coerce categorical variables into numerics if it needs to.

Finally, each assertion function has a counterpart that using standard
evaluation. The counterpart functions are postfixed by "_" (an underscore).

### More info

For more info, check out the `assertr` vignette

```r
    # vignette("assertr")
```
Or [read it here](http://cran.r-project.org/web/packages/assertr/vignettes/assertr.html)



## final exercise

_Announcement: some new 2016 data has been recorded for gapminder!_  
Let's check this data before we combine it with the official dataset:

**Step 1**: write down some assertions which pass for all of gapminder

**Step 2**: download [this file](SuppMatt/gapminder_2016.csv) and see if you can find all the errors!
