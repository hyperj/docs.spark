# By

## CREATE TABLE

### PARTITIONED BY

```sql
[PARTITIONED BY (col3 data_type [COMMENT col_comment], ...)]
```

### CLUSTERED/SORTED BY

```sql
[CLUSTERED BY (col1, ...) [SORTED BY (col1 [ASC|DESC], ...)] INTO num_buckets BUCKETS]
```

### SKEWED BY

```sql
[SKEWED BY (col1, col2, ...) ON ((col_value, col_value, ...), ...)
```

## Query

### ORDER BY

### CLUSTER BY

### DISTRIBUTE BY(repartition)

```properties
# The default number of partitions to use when shuffling data for joins or aggregations.
spark.sql.shuffle.partitions=200
```

### SORT BY

## WINDOW

### CLUSTER BY

### PARTITION|DISTRIBUTE BY

### ORDER|SORT BY

## Writer

### partitionBy

### bucketBy

### sortBy

## Reference