---
title: It Pays to Be Lazy
slug: polars-lazy
description: "A brief introduction Polars lazy mode"
summary: "A brief introduction Polars lazy mode"
categories: ["explainer"]
tags: ["polars", "data engineering", "python"]
date: 2026-02-12
draft: false
---

Polars is an open-source library built for working with tabular data that, over the last few years, has quickly caught up to pandas in terms of popularity.  A huge advantage of polars over pandas is its ability to process data more efficiently. In this post, we’ll cover one of the key mechanisms that enables polars to do this: lazy mode!

## What is lazy mode?

Polars provides two ways to work with data: 
1. **eager** -  immediately executing a query line-by-line; 
2. **lazy**  -  identifying which corners can be cut and then executing the query.

## How lazy mode creates a dataframe?

Imagine that we want to load the names of the director in lowercase from movies with `IMDB_Rating` greater than 8.0. the following example from the [imdb dataset](https://www.kaggle.com/datasets/harshitshankhdhar/imdb-dataset-of-top-1000-movies-and-tv-shows).
    
In eager mode,  our query would look like:

```python
eager_query = (
    pl.read_csv(f"./data/imdb.csv")
    .select(pl.col("IMDB_Rating"), pl.col("Director").str.to_lowercase())
    .filter(pl.col("IMDB_Rating") > 8.0)
)
```

And be executed as followed:

1. Open  the `imdb.csv` file
2. read in all 16 columns for all rows
3. transform the `Director` column to lowercase
4. apply a filter on the `imdb_score` column

In lazy mode , our query would look like:

```python
lazy_query = (
    pl.scan_csv(f"./data/imdb.csv")
    .select(pl.col("IMDB_Rating"), pl.col("Director").str.to_lowercase())
    .filter(pl.col("IMDB_Rating") > 8.0)
    .collect()
    )
```

And be executed as followed:

1. Open the`imdb.csv` file
2. apply the filter on the `IMDB_Rating` column **while** the CSV is being read in
3. read 2 columns for only the filter rows
4. transform the `Director` column to lowercase

Notice how we have to include the  `.collect()` method at the end of this query. This is because, unlike `.read_csv()` , the `.scan_csv()` method outputs a `LazyFrame` object, a representation of our query, which we then have to execute via the `.collect()` method to get a DataFrame.

We can visualise the optimised query plan as follows:

```python
lazy_query.explain(optimized=True)
```

Which gives the following output:

```xml
 WITH_COLUMNS:
 [col("Director").str.lowercase()]

    CSV SCAN data/imdb.csv
    PROJECT 2/16 COLUMNS
    SELECTION: [(col("IMDB_Rating")) > (8.0)]
```

## How does polars optimize a query?
  
We do not have to specify how to optimize the query, when we use the lazy API, Polars automatically run several [optimisations](https://docs.pola.rs/user-guide/lazy/optimizations/)  on our query. For example, in our above query polars noticed that it can limit the number of rows of data it needs load into a `DataFrame`  and store in the memory by applying the filter clause as it is reading in the CSV file (this is called *Predicate Pushdown).*
    
## What's the point of eager mode, then?
    
If lazy mode is so great, then why should I ever bother using eager mode? Eager mode is best for when we want full insight into how the query is being executed. So, we would likley want to make use of eager mode during development for debugging. Lazy mode really shines as we start working with bigger datasets, so we would likely  want to use a lazy query when we move into production and are want to optimise a script for speed.

## Closing Remarks
Pandas has long been the go-to python library for tabular data, it integrates well with other packages, it has a huge community around it and is embedded in many products, so it is unlikely to be completely replaced anytime soon. However, as we have seen here, polars’ lazy mode gives it a competitive edge over pandas for ETL and production, and would be well-worth exploring on your next data-driven project.