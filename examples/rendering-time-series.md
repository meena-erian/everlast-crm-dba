# Rendering Time Series
A time series graph is a graph where x-axis is **__time__**, and y-axis is the factor 
that the graph aims to visualize its relation with time AKA the **__value__** and in 
order to render multiple lines in the same graph, a unique **__metric__** is set for each.

The data of any graph is represented in connected points, and in time series graphs,
the time factor between each point and the next point in the graph must be constant,
becasue otherwise the resulted graph would be missleading.

Take the below query for example:

```sql
SELECT 
  invoice.invoicedate AS time,
  invoice.total AS value
FROM 
  accounting.invoice
```

| time | value |
|------|-------|
| 2020-1-1 | 1000 |
| 2020-1-1 | 2000 |
| 2020-1-2 | 50 |
| 2020-1-4 | 2000 |

The above example data generated from the above query is **NOT** a valid 
data set for a time serise graph because:
1. it shows multiple records for the same time '2020-1-1'
2. the time difference between each sample is not constant, as it's 
only one day between the 2nd and 3rd rows, while there's a gape after that 
making two days difference between the 3rd and 4th row.

In order to fix this, we must group these results based on time.
in PostgreSQL you can do this using the **generate_series** function.

First, we will have to generate the blank series like this:

```sql
SELECT 
  generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 day'::interval
    ) AS time,
0 AS value
```

This will generate the following results:

| time | value |
|------|-------|
| 2020-1-1 | 0 |
| 2020-1-2 | 0 |
| 2020-1-3 | 0 |
| 2020-1-4 | 0 |

There's no data in the above query results, however, 
it's a valid data set for time series graphs because:
1. There are no time gapes (it shows results for each and every day in the range)
2. There are no duplicates (it shows only one row for each day)
3. The sequence shows equally spaced points (one day between each result and the next one)

Next step is that we will need to input the data into the blank time serise. 
We will have to make nested queries in order to achieve that:


