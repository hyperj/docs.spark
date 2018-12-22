# Hint

```antlrv4
hint
    : '/*+' hintStatements+=hintStatement (','? hintStatements+=hintStatement)* '*/'
    ;

hintStatement
    : hintName=identifier
    | hintName=identifier '(' parameters+=primaryExpression (',' parameters+=primaryExpression)* ')'
    ;
```

- HINT

```sql
SELECT /*+ HINT(t) */ *
  FROM t
```

- BROADCASTJOIN

```sql
SELECT /*+ BROADCASTJOIN(b) */ *
  FROM T1 a JOIN T2 b ON a.key = b.key
```

- MAPJOIN

```sql
SELECT /*+ MAPJOIN(b) */ *
  FROM T1 a JOIN T2 b ON a.key = b.key
```

- STREAMTABLE

```sql
SELECT /*+ STREAMTABLE(b) */ *
 FROM T1 a JOIN T2 b ON a.key = b.key
```

- INDEX

```sql
SELECT /*+ INDEX(t, ix_job_name) */ * 
  FROM t
```

- COALESCE

```sql
SELECT /*+ COALESCE(200) */ *
  FROM t
```

- REPARTITION

```sql
SELECT /*+ REPARTITION(200) */ *
  FROM t
```

## Reference