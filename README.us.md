# SQL Optimization

### Optimize queries based on the query optimization guidelines

Follow the SQL best practices to ensure query optimization:
1. Index all the predicates in JOIN, WHERE, ORDER BY and GROUP BY clauses.
WebSphere Commerce typically depends heavily on indexes to improve SQL performance and scalability. Without proper indexes, SQL queries can cause table scans, which causes either performance or locking problems. It is recommended that all predicate columns be indexed. The exception being where column data has very low cardinality.

2. Avoid using functions in predicates.
   The index is not used by the database if there is a function on the column. For example:

   ```sql
   SELECT * FROM TABLE1 WHERE UPPER(COL1)='ABC'Copy
   ```

   As a result of the function UPPER(), the index on COL1 is not used by database optimizers. OracleIf the function cannot be avoided in the SQL, you need to create a function-based index in Oracle or generated columns in DB2 to improve performance.

3. Avoid using wildcard (%) at the beginning of a predicate.
      The predicate LIKE '%abc' causes full table scan. For example:
      ```sql 
      SELECT * FROM TABLE1 WHERE COL1 LIKE '%ABC'Copy
      ```
      
      This is a known performance limitation in all databases.

4. Avoid unnecessary columns in SELECT clause.
   Specify the columns in the SELECT clause instead of using SELECT *. The unnecessary columns places extra loads on the database, which slows down not just the single SQL, but the whole system.

5. Use inner join, instead of outer join if possible.
   The outer join should only be used if it is necessary. Using outer join limits the database optimization options which typically results in slower SQL execution.

6. DISTINCT and UNION should be used only if it is necessary.
   DISTINCT and UNION operators cause sorting, which slows down the SQL execution. Use UNION ALL instead of UNION, if possible, as it is much more efficient.

7. Oracle: Oracle 10g and 11g requires that the CLOB/BLOB columns must be put at the end of the statements.
   Otherwise, it causes failure when the input value size is larger than 1000 characters.

8. The ORDER BY clause is mandatory in SQL if the sorted result set is expected.
   The ORDER BY keyword is used to sort the result-set by specified columns. Without the ORDER BY clause, the result set is returned directly without any sorting. The order is not guaranteed. Be aware of the performance impact of adding the ORDER BY clause, as the database needs to sort the result set, resulting in one of the most expensive operations in SQL execution.

### Push predicates into the OUTER JOIN clause whenever possible
For SQL queries with the LEFT OUTER JOIN, pushing predicates of the right table from the WHERE clause into the ON condition helps the database optimizer generate a more efficient query. Predicates of the left table can stay in the WHERE clause.

Similarly, for the SQL queries with the RIGHT OUTER JOIN, predicates for the right table should be moved from the WHERE clause into the ON condition.

For example, the suboptimal query is rewritten by pushing predicates applicable to the table TAB_B into the ON clause. The TAB_A specific predicates in the WHERE clause can either stay, or be pushed into the ON clause:

Suboptimal SQL statement:

```sql
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 WHERE TAB_A.COL1=123 AND TAB_B.COL2=456
```

Optimized SQL statement:

```sql
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 AND TAB_B.COL2=456 WHERE TAB_A.COL1=123
```

Predicates for any INNER joins can stay in the WHERE clause. If tables TAB_A and TAB_B are defined as views, the optimizer can push these predicates into the views.

### Duplicate constant condition for different tables whenever possible
When two tables, A and B, are joined and there is a constant predicate on one of the joined columns, for example, A.id=B.id and A.id in (10, 12), the constant predicate should be duplicated for the joined column of the second table. That is, A.id=B.id and A.id in (10, 12) and B.id in (10, 12).

For example, TAB_A has a LEFT OUTER JOIN relationship with TAB_B. If there is a TAB_A specific conditions and a cross table condition with TAB_B, create an extra TAB_B specific condition based on TAB_A requirement and keep cross table conditions in the ON clause:

Suboptimal SQL statement:

```SQL
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 WHERE TAB_A.COL1 IN (123, 456) AND TAB_B.COL2=TAB_A.COL1
```

Optimized SQL statement:

```SQL
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 AND TAB_B.COL2 IN (123, 456) AND TAB_B.COL2=TAB_A.COL1  WHERE TAB_A.COL1 IN (123, 456)
```


In particular, if the constant predicate has only 1 value (e.g COL1=123), the second predicate should be converted to a constant predicate too.

For example:

Suboptimal SQL statement:

```sql
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 WHERE TAB_A.COL1=123 AND TAB_B.COL2=TAB_A.COL1
```


Optimized SQL statement:

```sql
SELECT TAB_A.COL1, TAB_B.COL1 FROM TAB_A LEFT OUTER JOIN TAB_B ON TAB_A.COL3 = TAB_B.COL3 AND TAB_B.COL2=123 WHERE TAB_A.COL1=123
```

Using nested table definitions to replace workspaces views
To improve performance in the workspaces environment you can use nested table definitions to replace the standard views defined for the content-managed tables. The following example query uses the default views for tables CATENTRY, ATTRIBUTE, and ATTRVALUE:

```sql
SELECT
     CATENTRY.CATENTRY_ID,  ATTRIBUTE.ATTRIBUTE_ID, ATTRIBUTE.NAME, ATTRIBUTE.LANGUAGE_ID, ATTRIBUTE.CATENTRY_ID,
     ATTRVALUE.ATTRVALUE_ID,  ATTRVALUE.ATTRIBUTE_ID, ATTRVALUE.CATENTRY_ID, ATTRVALUE.STRINGVALUE
FROM CATENTRY, ATTRVALUE, ATTRIBUTE
WHERE
     ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
     AND CATENTRY.CATENTRY_ID IN (10683)
     AND ATTRVALUE.LANGUAGE_ID IN ( -1,-2,-3,-4,-5 ) 
     AND ATTRIBUTE.LANGUAGE_ID=ATTRVALUE.LANGUAGE_ID 
     AND ATTRIBUTE.ATTRIBUTE_ID=ATTRVALUE.ATTRIBUTE_ID
ORDER BY 
     ATTRIBUTE.SEQUENCECopy

```

This query can be rewritten to use nested tables definitions for tables CATENTRY, ATTRVALUE, and ATTRIBUTE:

```sql
SELECT
     CATENTRY.CATENTRY_ID,  ATTRIBUTE.ATTRIBUTE_ID, ATTRIBUTE.NAME, ATTRIBUTE.LANGUAGE_ID, ATTRIBUTE.CATENTRY_ID,
     ATTRVALUE.ATTRVALUE_ID,  ATTRVALUE.ATTRIBUTE_ID, ATTRVALUE.CATENTRY_ID, ATTRVALUE.STRINGVALUE
FROM (
          SELECT 
               CATENTRY_ID
          FROM 
               DB2INST1.CATENTRY
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.CATENTRY
                              WHERE DB2INST1.CATENTRY.CATENTRY_ID = WCW101.CATENTRY.CATENTRY_ID) 
          UNION ALL 
          SELECT
               CATENTRY_ID
          FROM
               WCW101.CATENTRY
          WHERE WCW101.CATENTRY.CONTENT_STATUS <> 'D'
     ) CATENTRY,
     (
          SELECT  
               ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
          FROM 
               DB2INST1.ATTRVALUE 
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.ATTRVALUE 
                              WHERE DB2INST1.ATTRVALUE.ATTRVALUE_ID = WCW101.ATTRVALUE.ATTRVALUE_ID) 
          UNION ALL 
          SELECT
               ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
          FROM
               WCW101.ATTRVALUE 
          WHERE WCW101.ATTRVALUE.CONTENT_STATUS <> 'D'
     ) ATTRVALUE,  
     (
          SELECT 
               ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
          FROM 
               DB2INST1.ATTRIBUTE 
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.ATTRIBUTE
                              WHERE DB2INST1.ATTRIBUTE.ATTRIBUTE_ID = WCW101.ATTRIBUTE.ATTRIBUTE_ID) 
          UNION ALL 
          SELECT
               ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
          FROM
               WCW101.ATTRIBUTE
          WHERE WCW101.ATTRIBUTE.CONTENT_STATUS <> 'D'
      )  ATTRIBUTE
WHERE
     ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
     AND CATENTRY.CATENTRY_ID IN (10683)
     AND ATTRVALUE.LANGUAGE_ID IN ( -1,-2,-3,-4,-5 ) 
     AND ATTRIBUTE.LANGUAGE_ID=ATTRVALUE.LANGUAGE_ID 
     AND ATTRIBUTE.ATTRIBUTE_ID=ATTRVALUE.ATTRIBUTE_ID
ORDER BY 
     ATTRIBUTE.SEQUENCECopy
```



Occasionally, further performance gains can be achieved by pushing in predicates into nested table definitions. This might reduce the amount of data that needs to be fetched for nested tables. Continuing with the preceding example, the predicate LANGUAGE_ID IN ( -1,-2,-3,-4,-5 ) is injected into the definition of the ATTRVALUE nested table in the following query:

```sql
SELECT
     CATENTRY.CATENTRY_ID,  ATTRIBUTE.ATTRIBUTE_ID, ATTRIBUTE.NAME, ATTRIBUTE.LANGUAGE_ID, ATTRIBUTE.CATENTRY_ID,
     ATTRVALUE.ATTRVALUE_ID,  ATTRVALUE.ATTRIBUTE_ID, ATTRVALUE.CATENTRY_ID, ATTRVALUE.STRINGVALUE
FROM (
          SELECT 
               CATENTRY_ID
          FROM 
               DB2INST1.CATENTRY
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.CATENTRY
                              WHERE DB2INST1.CATENTRY.CATENTRY_ID = WCW101.CATENTRY.CATENTRY_ID) 
          UNION ALL 
          SELECT
               CATENTRY_ID
          FROM
               WCW101.CATENTRY
          WHERE WCW101.CATENTRY.CONTENT_STATUS <> 'D'
     ) CATENTRY,
     (
          SELECT 
               ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
          FROM 
               DB2INST1.ATTRVALUE 
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.ATTRVALUE 
                              WHERE DB2INST1.ATTRVALUE.ATTRVALUE_ID = WCW101.ATTRVALUE.ATTRVALUE_ID)
               AND DB2INST1.ATTRVALUE.LANGUAGE_ID IN ( -1,-2,-3,-4,-5 )
          UNION ALL 
          SELECT
               ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
          FROM
               WCW101.ATTRVALUE 
          WHERE WCW101.ATTRVALUE.CONTENT_STATUS <> 'D' 
               AND WCW101.ATTRVALUE.LANGUAGE_ID IN ( -1,-2,-3,-4,-5 ) 
     ) ATTRVALUE,  
     (
          SELECT 
               ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
          FROM 
               DB2INST1.ATTRIBUTE 
          WHERE  NOT EXISTS (
                              SELECT '1' 
                              FROM WCW101.ATTRIBUTE
                              WHERE DB2INST1.ATTRIBUTE.ATTRIBUTE_ID = WCW101.ATTRIBUTE.ATTRIBUTE_ID) 
          UNION ALL 
          SELECT
               ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
          FROM
               WCW101.ATTRIBUTE
     WHERE WCW101.ATTRIBUTE.CONTENT_STATUS <> 'D'
     )  ATTRIBUTE
WHERE
     ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
     AND CATENTRY.CATENTRY_ID IN (10683)	
     AND ATTRIBUTE.LANGUAGE_ID=ATTRVALUE.LANGUAGE_ID 
     AND ATTRIBUTE.ATTRIBUTE_ID=ATTRVALUE.ATTRIBUTE_ID
ORDER BY 
     ATTRIBUTE.SEQUENCECopy
```


Queries using nested table definitions instead of views need to explicitly reference the base and the write schemas. These queries are only applicable to the workspaces environment, and, therefore, need to be defined in a query template under a special content management section ('cm'). The following is an example of the preceding query, defined as a query template for both the runtime and the workspaces environments:
**Note**
The following snippet uses $CM:READ$, $CM:WRITE$, and $CM:BASE$ tags. They will be replaced at runtime by the READ, WRITE and BASE schema names when the SQL is executed by the Data Service Layer.
See Workspaces data model for more information.

```mysql
 BEGIN_XPATH_TO_SQL_STATEMENT
     name=/CatalogEntry[CatalogEntryIdentifier[(UniqueID=)]]+IBM_Attributes
     base_table=CATENTRY
 sql=
      SELECT CATENTRY.$COLS:CATENTRY_ID$, ATTRIBUTE.$COLS:ATTRIBUTE$,ATTRVALUE.$COLS:ATTRVALUE$ 
      FROM CATENTRY, ATTRVALUE, ATTRIBUTE
      WHERE
           ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
           AND CATENTRY.CATENTRY_ID IN (?UniqueID?)
           AND ATTRVALUE.LANGUAGE_ID IN  ($CONTROL:LANGUAGES$) 
           AND ATTRIBUTE.LANGUAGE_ID=ATTRVALUE.LANGUAGE_ID 
           AND ATTRIBUTE.ATTRIBUTE_ID=ATTRVALUE.ATTRIBUTE_ID
      ORDER BY 
           ATTRIBUTE.SEQUENCE

 <!-- the following query is optimized for workspaces -->
 cm
 sql=
      SELECT CATENTRY.$COLS:CATENTRY_ID$, ATTRIBUTE.$COLS:ATTRIBUTE$,ATTRVALUE.$COLS:ATTRVALUE$ 
      FROM (
       SELECT 
           CATENTRY_ID
        FROM 
           $CM:BASE$.CATENTRY
        WHERE  NOT EXISTS (
                          SELECT '1' 
                          FROM $CM:WRITE$.CATENTRY
                          WHERE $CM:BASE$.CATENTRY.CATENTRY_ID = $CM:WRITE$.CATENTRY.CATENTRY_ID) 
      UNION ALL 
      SELECT
           CATENTRY_ID
      FROM
           $CM:WRITE$.CATENTRY
      WHERE $CM:WRITE$.CATENTRY.CONTENT_STATUS <> 'D'
 ) CATENTRY,
 (
      SELECT 
           ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
      FROM 
           $CM:BASE$.ATTRVALUE 
      WHERE  NOT EXISTS (
                          SELECT '1' 
                          FROM $CM:WRITE$.ATTRVALUE 
                          WHERE $CM:BASE$.ATTRVALUE.ATTRVALUE_ID = $CM:WRITE$.ATTRVALUE.ATTRVALUE_ID)
             AND $CM:BASE$.ATTRVALUE.LANGUAGE_ID IN ($CONTROL:LANGUAGES$)
      UNION ALL 
      SELECT
           ATTRVALUE_ID,LANGUAGE_ID,CATENTRY_ID,ATTRIBUTE_ID,STRINGVALUE
      FROM
           $CM:WRITE$.ATTRVALUE 
      WHERE $CM:WRITE$.ATTRVALUE.CONTENT_STATUS <> 'D' 
             AND $CM:WRITE$.ATTRVALUE.LANGUAGE_ID IN ($CONTROL:LANGUAGES$) 
 ) ATTRVALUE,  
 (
      SELECT 
           ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
      FROM 
           $CM:BASE$.ATTRIBUTE 
      WHERE  NOT EXISTS (
                          SELECT '1' 
                          FROM $CM:WRITE$.ATTRIBUTE
                          WHERE $CM:BASE$.ATTRIBUTE.ATTRIBUTE_ID = $CM:WRITE$.ATTRIBUTE.ATTRIBUTE_ID) 
      UNION ALL 
      SELECT
           ATTRIBUTE_ID,LANGUAGE_ID,CATENTRY_ID,SEQUENCE,NAME
      FROM
           $CM:WRITE$.ATTRIBUTE
      WHERE $CM:WRITE$.ATTRIBUTE.CONTENT_STATUS <> 'D'
   )  ATTRIBUTE
 WHERE
      ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
      AND CATENTRY.CATENTRY_ID IN (?UniqueID?)	
      AND ATTRIBUTE.LANGUAGE_ID=ATTRVALUE.LANGUAGE_ID 
      AND ATTRIBUTE.ATTRIBUTE_ID=ATTRVALUE.ATTRIBUTE_ID
 ORDER BY 
      ATTRIBUTE.SEQUENCE
     
 END_XPATH_TO_SQL_STATEMENT
```


Splitting queries
If applying these techniques did not provide the necessary performance improvements, you might consider splitting queries joining many tables into multiple queries. The objective of splitting the query is to minimize the number of views used in the join. If you are splitting a single-step query, you will need to convert it to a two-step query with one or more association queries. When splitting one of the association SQL statements for a two-step query, you need to add additional association SQL statements to the access profile definition. When queries are split, you should use inner joins instead of outer joins whenever possible.

Be cautious when splitting queries. If you take this approach too extensively, it might result in many queries, each selecting from one table only. In this case, you will need to perform table joins in memory by writing Java code, for example, using a graph composer. In some cases, you will need to feed the result of one query into another. These types of queries should be considered as a last resort, as maintainability, customization and migration challenges can arise.

Oracle
Using common expression syntax in Oracle
The Oracle optimizer typically generates very efficient queries, where a temporary table is created based on the result set of the original query joining views. A temporary table can be created using the common table expression syntax.

For example, the following query fetching product attribute values can run significantly faster after rewriting it using the common table expression syntax:

Standard query fetching product attributes:
BEGIN_ASSOCIATION_SQL_STATEMENT

name=IBM_CatalogEntryAttributeValue
base_table=CATENTRY

sql =
SELECT 
      CATENTRY.$COLS:CATENTRY$, ATTRVALUE.$COLS:ATTRVALUE$, 
      ATTRVALUE2.$COLS:ATTRVALUE$
FROM CATENTRY, ATTRVALUE 
JOIN ATTRIBUTE 
      ON ATTRIBUTE.LANGUAGE_ID = ATTRVALUE.LANGUAGE_ID 
      AND ATTRIBUTE.ATTRIBUTE_ID = ATTRVALUE.ATTRIBUTE_ID 
LEFT OUTER JOIN ATTRVALUE ATTRVALUE2 
      ON ATTRIBUTE.ATTRIBUTE_ID = ATTRVALUE2.ATTRIBUTE_ID 
      AND ATTRVALUE2.CATENTRY_ID = 0 
      AND ATTRIBUTE.LANGUAGE_ID = ATTRVALUE2.LANGUAGE_ID 
WHERE CATENTRY.CATENTRY_ID IN ( $ENTITY_PKS$) 
      AND ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
      AND ATTRVALUE.LANGUAGE_ID IN ($CTX:LANG_ID$) 

END_ASSOCIATION_SQL_STATEMENTCopy
Using common expression syntax:
BEGIN_ASSOCIATION_SQL_STATEMENT

name=IBM_CatalogEntryAttributeValue
base_table=CATENTRY

sql =
WITH TEMP_TABLE AS (
SELECT 
      CATENTRY.CE_$COLS:CATENTRY$, ATTRVALUE.ATTR_$COLS:ATTRVALUE$, 
      ATTRVALUE2.ATTR2_$COLS:ATTRVALUE$
FROM CATENTRY, ATTRVALUE 
JOIN ATTRIBUTE 
      ON ATTRIBUTE.LANGUAGE_ID = ATTRVALUE.LANGUAGE_ID 
      AND ATTRIBUTE.ATTRIBUTE_ID = ATTRVALUE.ATTRIBUTE_ID 
LEFT OUTER JOIN ATTRVALUE ATTRVALUE2 
      ON ATTRIBUTE.ATTRIBUTE_ID = ATTRVALUE2.ATTRIBUTE_ID 
      AND ATTRVALUE2.CATENTRY_ID = 0 
      AND ATTRIBUTE.LANGUAGE_ID = ATTRVALUE2.LANGUAGE_ID 
WHERE CATENTRY.CATENTRY_ID IN ( $ENTITY_PKS$) 
      AND ATTRVALUE.CATENTRY_ID = CATENTRY.CATENTRY_ID 
      AND ATTRVALUE.LANGUAGE_ID IN ($CTX:LANG_ID$) 
) SELECT * FROM TEMP_TABLE

END_ASSOCIATION_SQL_STATEMENTCopy
OracleThis technique enables the Oracle optimizer to push the necessary predicates into the views. As more data is filtered out in the early stages of query processing, subsequent joins will need to be applied to a much smaller data set. This often results in significant performance improvements.
