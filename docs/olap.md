# OLAP

```antlrv4
aggregation
    : GROUP BY groupingExpressions+=expression (',' groupingExpressions+=expression)* (
      WITH kind=ROLLUP
    | WITH kind=CUBE
    | kind=GROUPING SETS '(' groupingSet (',' groupingSet)* ')')?
    | GROUP BY kind=GROUPING SETS '(' groupingSet (',' groupingSet)* ')'
    ;

groupingSet
    : '(' (expression (',' expression)*)? ')'
    | expression
    ;
```

## GROUPING SETS/ROLLUP/CUBE

Parsed Logical Plan -> Expand(mapPartitions)、Aggregate(Hash#mapPartitionsWithIndex)

- 其中 ROLLUP / CUBE 为 GROUPING SETS 的特例
- ROLLUP 用于支持上卷和下钻的情况的简易写法
- CUBE 用于支持构建立方体的情况的简易写法

## Reference
