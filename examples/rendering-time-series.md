# Rendering Time Series
A time series graph is a graph where x-axis is **__time__**, and y-axis is the factor 
that the graph aims to visualize its relation with time AKA the **__value__** and in 
order to render multiple lines in the same graph, a unique **__metric__** is set for each.

The data of any graph is represented in connected points, and in time series graphs,
the time factor between each point and the next point in the graph must be constant,
because otherwise the resulted graph would be misleading.

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
data set for a time series graph because:
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

Next step is that we will need to input the data into the blank time series. 
We will have to join this empty table with our data table:

```sql
SELECT  *
FROM
generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 day'::interval
    ) AS sample_time
LEFT JOIN accounting.invoice
ON invoice.invoicedate = sample_time
```
Now the above query will generate the following results:

| sample_time | invoicedate | total |
|-------------|-------------|-------|
| 2020-1-1 | 2020-1-1 | 2000 |
| 2020-1-1 | 2020-1-1 | 1000 |
| 2020-1-2 | 2020-1-2 | 50   |
| 2020-1-3 | | |
| 2020-1-4 | 2020-1-4 | 2000 |

The first column in this result was generated from generate_series and 
the other two invoice columns were joined to it based on the date.

The above results meets the 1st and the 3rd conditions for a time series 
data set described above (1. There are no time gapes, 3. The sequence shows
equally spaced points)
However, it doesn't meet the second requirement because there are multiple 
invoices on the same date. (2020-1-1)

in order to fix this, we will have to sum invoices in the same time into one total, just like this:

```sql
SELECT
  sample_time as time,
  SUM(total) as value
FROM
generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 day'::interval
    ) AS sample_time
LEFT JOIN accounting.invoice
ON invoice.invoicedate = sample_time
GROUP BY sample_time
```

| time | value |
|------|-------|
| 2020-1-1 | 3000 |
| 2020-1-2 | 50 |
| 2020-1-3 |  |
| 2020-1-4 | 2000 |

Now the above query shows results that meet the three conditions of a time series data set. 
(notice how the first to invoices with values 1000, and 2000 were joined into one sample)

Before we render this, we also need to add zeros for empty samples. We can easily do this using
the COALESCE function.

```sql
SELECT
  sample_time as time,
  SUM(COALESCE(total, 0)) as value
FROM
generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 day'::interval
    ) AS sample_time
LEFT JOIN accounting.invoice
ON invoice.invoicedate = sample_time
GROUP BY sample_time
```

The COALESCE function replaces empty cells with a user defined value, in this case it's zero.

| time | value |
|------|-------|
| 2020-1-1 | 3000 |
| 2020-1-2 | 50 |
| 2020-1-3 | 0 |
| 2020-1-4 | 2000 |

The above results shows a perfect time series data set, 
however, the query itself isn't perfect. it will only work fine in the 
provided invoice data because the sequence is one day and the date value 
is a date with no time (year-month-day no hour, minute or second)
but usually the time field is provided as a continuous value.
For example if the invoice data was like this:

| time | value |
|------|-------|
| 2020-1-1 | 1000 |
| 2020-1-1T5:30 | 2000 |
| 2020-1-2 | 50 |
| 2020-1-4 | 2000 |

Where the only difference is that the second invoice is at 5:30 AM rather than 00:00, 
in this case, the above query will filter out the second invoice showing only the 
value 1000 for day 2020-1-1 because the second invoice isn't an identical match. for any sample!

To prevent this from happening, we have to **truncate** the time to the nearest sequence unit (1 day in this case)
we can do this using the **date_trunc** function provided by postgreSQL.

```sql
SELECT
  sample_time as time,
  SUM(COALESCE(total, 0)) as value
FROM
generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 day'::interval
    ) AS sample_time
LEFT JOIN accounting.invoice
ON date_trunc('day', invoice.invoicedate ) = sample_time
GROUP BY sample_time
```

This time series should render a graph like this:

![revenue graph](/media/graph0x1.png)


And of course, the time squence doesn't need to be a day, 
for example if you need to render a revenue per houre graph, 
the query will look like this:

```sql
SELECT
  sample_time as time,
  SUM(COALESCE(total, 0)) as value
FROM
generate_series
    (
      ('2020-1-1') :: timestamp,
      ('2020-1-4') :: timestamp,
      '1 hour'::interval
    ) AS sample_time
LEFT JOIN accounting.invoice
ON date_trunc('hour', invoice.invoicedate ) = sample_time
GROUP BY sample_time
```
Notice how we have not changed anything except for the interval in **generate_series** and in **date_trunc**
