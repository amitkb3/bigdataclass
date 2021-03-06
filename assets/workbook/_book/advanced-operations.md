

# Advanced Operations



## Simple wrapper function
*Create a function that accepts a value that is passed to a specific dplyr operation*


1. The following `dplyr` operation is fixed to only return the mean of *arrtime*.  The desire is to create a function that returns the mean of any variable passed to it.

```r
flights %>%
  summarise(mean = mean(arrtime, na.rm = TRUE))
```

2. Load the `rlang` library, and create a function with one argument. The function will simply return the result of `equo()`


```r
library(rlang)

my_mean <- function(x){
  x <- enquo(x)
  x
}

my_mean(mpg)
```

3. Add the `summarise()` operation, and replace *arrtime* with *!! x*

```r
library(rlang)

my_mean <- function(x){
  x <- enquo(x)
  flights %>%
    summarise(mean = mean(!! x, na.rm = TRUE))
}
```

4. Test the function with *deptime*

```r
my_mean(deptime)
```

5. Make the function use what is passed to the *x* argument as the name of the calculation.  Replace *mean = * with *!! quo_name(x) :=* .

```r
my_mean <- function(x){
  x <- enquo(x)
  flights %>%
    summarise(!! quo_name(x) := mean(!! x, na.rm = TRUE))
  
}
```

6. Test the function again with *arrtime*.  The name of the variable should now by *arrtime*

```r
my_mean(arrtime)
```

7. Test the function with a formula: *arrtime+deptime*.

```r
my_mean(arrtime+deptime)
```

8. Make the function generic by adding a *.data* argument and replacing *flights* with *.data*

```r
my_mean <- function(.data, x){
  x <- enquo(x)
  .data %>%
    summarise(!! quo_name(x) := mean(!! x, na.rm = TRUE))
  
}
```

9. The function now behaves more like a `dplyr` verb. Start with *flights* and pipe into the function.

```r
flights %>%
  my_mean(arrtime)
```

10. Test the function with a different data set.  Use `mtcars` and *mpg* as the *x* argument.

```r
mtcars %>%
  my_mean(mpg)
```

11. Clean up the function by removing the pipe

```r
my_mean <- function(.data, x){
  x <- enquo(x)
  summarise(
    .data, 
    !! quo_name(x) := mean(!! x, na.rm = TRUE)
  )
}
```

12. Test again, no visible changes should be there for the results

```r
mtcars %>%
  my_mean(mpg)
```

13. Because the function only uses `dplyr` operations, `show_query()` should work

```r
flights %>%
  my_mean(arrtime) %>%
  show_query()
```


## Multiple variables
*Create functions that handle a variable number of arguments. The goal of the exercise is to create an "anti-select()" function.*

1. Use *...* as the second argument of a function called `de_select()`.  Inside the function use `enquos()` to parse it

```r
de_select <- function(.data, ...){
  vars <- enquos(...)
  vars
}
```

2. Test the function using *airports*

```r
airports %>%
  de_select(airport, airportname)
```

3. Add a step to the function that iterates through each quosure and prefixes a minus sign to tell `select()` to drop that specific field.  Use `map()` for the iteration, and `expr()` to create the prefixed expression.

```r
de_select <- function(.data, ...){
  vars <- enquos(...)
  vars <- map(vars, ~ expr(- !! .x))
  vars
}
```

4. Run the same test to view the new results


```r
airports %>%
  de_select(airport, airportname)
```

5. Add the `select()` step.  Use *!!!* to parse the *vars* variable inside `select()`


```r
de_select <- function(.data, ...){
  vars <- enquos(...)
  vars <- map(vars, ~ expr(- !! .x))
  select(
    .data,
    !!! vars
  )
}
```

6. Run the test again, this time the operation will take place.  


```r
airports %>%
  de_select(airport, airportname)
```

7. Add a `show_query()` step to see the resulting SQL


```r
airports %>%
  de_select(airport, airportname) %>%
  show_query()
```

8. Test the function with a different data set, such as `mtcars`


```r
mtcars %>%
  de_select(mpg, wt, am)
```

## Multiple queries
*Suggested approach to avoid passing multiple, and similar, queries to the database*

1. Create a simple `dplyr` piped operation that returns the mean of *arrdelay* for the months of January, February and March as a group.


```r
flights %>%
  filter(month %in% c(1,2,3)) %>%
  summarise(mean = mean(arrdelay, na.rm = TRUE)) 
```

2. Assign the first operation to a variable called *a*, and create copy of the operation but changing the selected months to January, March and April.  Assign the second one to a variable called *b*.


```r
a <- flights %>%
  filter(month %in% c(1,2,3)) %>%
  summarise(mean = mean(arrdelay, na.rm = TRUE)) 

b <- flights %>%
  filter(month %in% c(1,3,4)) %>%
  summarise(mean = mean(arrdelay, na.rm = TRUE)) 
```

3. Use *union()* to pass *a* and *b* at the same time to the database.


```r
union(a, b)
```

4. Assign to a new variable called *months* an overlapping set of months.  


```r
months <- list(
  c(1,2,3),
  c(1,3,4),
  c(2,4,6)
)
```

5. Use `map()` to cycle through each set of overlapping months.  Notice that it returns three separate results, meaning that it went to the database three times.


```r
months %>%
  map( ~ flights %>%
         filter(month %in% .x) %>%
         summarise(mean = mean(arrdelay, na.rm = TRUE)) 
  )
```

6. Add a `reduce()` operation and use `union()` command to create a single query.


```r
months %>%
  map( ~ flights %>%
         filter(month %in% .x) %>%
         summarise(mean = mean(arrdelay, na.rm = TRUE)) 
  ) %>%
  reduce(function(x, y) union(x, y))
```

7. Use `show_query()` to see the resulting single query sent to the database.


```r
months %>%
  map( ~ flights %>%
         filter(month %in% .x) %>%
         summarise(mean = mean(arrdelay, na.rm = TRUE)) 
  ) %>%
  reduce(function(x, y) union(x, y)) %>%
  show_query()
```


## Multiple queries with an overlaping range

1. Create a table with a *from* and *to* ranges.


```r
ranges <- tribble(
  ~ from, ~to, 
       1,   4,
       2,   5,
       3,   7
)
```

2. See how `map2()` works by passing the two variables as the *x* and *y* arguments, and adding them as the function. 


```r
map2(ranges$from, ranges$to, ~.x + .y)
```

3. Replace *x + y* with the `dplyr` operation from the previous exercise.  In it, re-write the filter to use *x* and *y* as the month ranges 


```r
map2(
  ranges$from, 
  ranges$to,
  ~ flights %>%
      filter(month >= .x & month <= .y) %>%
      summarise(mean = mean(arrdelay, na.rm = TRUE)) 
)
```

4. Add the reduce operation


```r
map2(
  ranges$from, 
  ranges$to,
  ~ flights %>%
      filter(month >= .x & month <= .y) %>%
      summarise(mean = mean(arrdelay, na.rm = TRUE)) 
) %>%
  reduce(function(x, y) union(x, y))
```

5. Add a `show_query()` step to see how the final query was constructed.


```r
map2(
  ranges$from, 
  ranges$to,
  ~ flights %>%
      filter(month >= .x & month <= .y) %>%
      summarise(mean = mean(arrdelay, na.rm = TRUE)) 
) %>%
  reduce(function(x, y) union(x, y)) %>%
  show_query()
```


