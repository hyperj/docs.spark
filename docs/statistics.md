# Statistics

Estimates of various statistics. The default estimation logic simply lazily multiplies the corresponding statistic produced by the children. 

## Statistics -> CatalogStatistics

- sizeInBytes: Physical size in bytes. For leaf operators this defaults to 1, otherwise it defaults to the product of children's `sizeInBytes`
- rowCount: Estimated number of rows
- attributeStats: Statistics for Attributes
- hints: Query hints

## ColumnStat -> CatalogColumnStat

- distinctCount: number of distinct values
- min: minimum value
- max: maximum value
- nullCount: number of nulls
- avgLen: average length of the values
- maxLen: maximum length of the values
- histogram: histogram of the values

## Histogram[HistogramBin]

- height: number of rows in each bin
- bins: equi-height histogram bins
  - lo: lower bound of the value range in this bin
  - hi: higher bound of the value range in this bin
  - ndv: approximate number of distinct values in this bin

## HintInfo

- broadcast
- join/shuffle

## [DataFrameStatFunctions](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/DataFrameStatFunctions.html)

Statistic functions for DataFrames.(Since: 1.4.0)

- approxQuantile: Calculates the approximate quantiles of numerical columns of a DataFrame
- bloomFilter: Builds a Bloom filter over a specified column
- corr: Calculates the Pearson Correlation Coefficient of two columns of a DataFrame
- countMinSketch: Builds a Count-min Sketch over a specified column
- cov: Calculate the sample covariance of two numerical columns of a DataFrame
- crosstab: Computes a pair-wise frequency table of the given columns
- freqItems: (Scala-specific) Finding frequent items for columns, possibly with false positives
- sampleBy: Returns a stratified sample without replacement based on the fraction given on each stratum

## Other 

### Dataset#describe

- count, mean, stddev, min, max
- StatFunctions.summary(ds, Seq("count", "mean", "stddev", "min", "25%", "50%", "75%", "max"))

### [Statistics](https://spark.apache.org/docs/latest/api/java/org/apache/spark/mllib/stat/Statistics.html)

API for statistical functions in MLlib

- colStats[MultivariateOnlineSummarizer]: column-wise summary statistics
- corr: Pearson correlation matrix
- chiSqTest: chi-squared test
- kolmogorovSmirnovTest: Kolmogorov-Smirnov test

## Reference

- [Spark SQL Package Summary](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/package-summary.html)