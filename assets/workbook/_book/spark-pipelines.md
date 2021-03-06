

#Spark pipelines



## Recreate the transformations 
*Overview of how most of the existing code will be reused*

1. Register a new table called *current* containing a sample of the base *flights* table

```r
model_data <- sdf_partition(
  tbl(sc, "flights"),
  training = 0.01,
  testing = 0.01,
  rest = 0.98
)
```

2. Recreate the `dplyr` code in the `cached_flights` variable from the previous unit

```r
pipeline_df <- model_data$training %>%
  mutate(
    arrdelay = ifelse(arrdelay == "NA", 0, arrdelay),
    depdelay = ifelse(depdelay == "NA", 0, depdelay)
  ) %>%
  select(
    month,
    dayofmonth,
    arrtime,
    arrdelay,
    depdelay,
    crsarrtime,
    crsdeptime,
    distance
  ) %>%
  mutate_all(as.numeric)
```

3. Create a new Spark pipeline

```r
flights_pipeline <- ml_pipeline(sc) %>%
  ft_dplyr_transformer(
    tbl = pipeline_df
  ) %>%
  ft_binarizer(
    input_col = "arrdelay",
    output_col = "delayed",
    threshold = 15
  ) %>%
  ft_bucketizer(
    input_col = "crsdeptime",
    output_col = "dephour",
    splits = c(0, 400, 800, 1200, 1600, 2000, 2400)
  ) %>%
  ft_r_formula(delayed ~ arrdelay + dephour) %>%
  ml_logistic_regression()

flights_pipeline
```

## Fit, evaluate, save


1. Fit (train) the pipeline's model

```r
model <- ml_fit(flights_pipeline, model_data$training)
model
```

2. Use the newly fitted model to perform predictions using `ml_transform()`

```r
predictions <- ml_transform(
  x = model,
  dataset = model_data$testing
)
```

3. Use `group_by()` to see how the model performed

```r
predictions %>%
  group_by(delayed, prediction) %>%
  tally()
```

4. Save the model into disk using `ml_save()`

```r
ml_save(model, "saved_model", overwrite = TRUE)
list.files("saved_model")
```
5. Save the pipeline using `ml_save()`

```r
ml_save(flights_pipeline, "saved_pipeline", overwrite = TRUE)
list.files("saved_pipeline")
```

6. Close the Spark session

```r
spark_disconnect(sc)
```
## Reload model

*Use the saved model inside a different Spark session*

1. Open a new Spark connection and reload the data

```r
library(sparklyr)
sc <- spark_connect(master = "local", version = "2.0.0")
spark_flights <- spark_read_csv(
  sc,
  name = "flights",
  path = "/usr/share/class/flights/data/",
  memory = FALSE,
  columns = file_columns,
  infer_schema = FALSE
)
```

2. Use `ml_load()` to reload the model directly into the Spark session

```r
reload <- ml_load(sc, "saved_model")
reload
```


3.  Create a new table called *current*. It needs to pull today's flights

```r
library(lubridate)

current <- tbl(sc, "flights") %>%
  filter(
    month == !! month(now()),
    dayofmonth == !! day(now())
  )

show_query(current)
```

4.  Create a new table called *current*. It needs to pull today's flights

```r
head(current)
```

5. Run predictions against the new data set

```r
new_predictions <- ml_transform(
  x = ml_load(sc, "saved_model"),
  dataset = current
)
```

6. Get a quick count of expected delayed flights

```r
new_predictions %>%
  summarise(late_fligths = sum(prediction, na.rm = TRUE))
```

## Reload pipeline
*Overview of how to use new data to re-fit the pipeline, thus creating a new pipeline model*

1. Use `ml_load()` to reload the pipeline into the Spark session

```r
flights_pipeline <- ml_load(sc, "saved_pipeline")
flights_pipeline
```

2. Create a new sample data set using `sample_frac()`

```r
sample <- tbl(sc, "flights") %>%
  sample_frac(0.001) 
```

3. Re-fit the model using `ml_fit()` and the new sample data

```r
new_model <- ml_fit(flights_pipeline, sample)
new_model
```

4. Save the newly fitted model 

```r
ml_save(new_model, "new_model", overwrite = TRUE)
list.files("new_model")
```

5. Disconnect from Spark

```r
spark_disconnect(sc)
```
