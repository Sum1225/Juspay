---
title: "Contained Database Collations"
description: "Learn how collation works in contained and non-contained databases. See issues that can arise when sessions cross between contained and non-contained contexts."
author: WilliamDAssafMSFT
ms.author: wiassaf
ms.date: 07/11/2023
ms.service: sql
ms.subservice: configuration
ms.topic: conceptual
helpviewer_keywords:
  - "contained database, collations"
---
# Contained database collations

 [!INCLUDE [SQL Server Azure SQL Managed Instance](../../includes/applies-to-version/sql-asdbmi.md)]

 Various properties affect the sort order and equality semantics of textual data, including case sensitivity, accent sensitivity, and the base language being used. These qualities are expressed to [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] through the choice of collation for the data. For a more in-depth discussion of collations themselves, see [Collation and Unicode Support](../../relational-databases/collations/collation-and-unicode-support.md).  
  
 Collations apply not only to data stored in user tables, but to all text handled by [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)], including metadata, temporary objects, variable names, etc. The handling of these differs in contained and non-contained databases. This change will not affect many users, but helps provide instance independence and uniformity. But this may also cause some confusion, as well as problems for sessions that access both contained and non-contained databases.  

 The collation behavior of contained databases differs subtly from that in non-contained databases. This behavior is generally beneficial, providing instance-independence and simplicity. Some users may have issues, particularly when a session accesses both contained and non-contained databases.  
  
 This article clarifies the content of the change, and examines areas where the change may cause problems.

> [!NOTE]
> For [!INCLUDE [ssazure-sqldb](../../includes/ssazure-sqldb.md)], collations for contained databases are different. The database collation and catalog collation can be set on database creation and cannot be updated. Specify a collation for data (`COLLATE`) and a catalog collation for system metadata and object identifiers (`CATALOG_COLLATION`). For more information, see [CREATE DATABASE](../../t-sql/statements/create-database-transact-sql.md?view=azuresqldb-current&preserve-view=true#collation_name).
  
## Non-contained databases
 All databases have a default collation (which can be set when creating or altering a database. This collation is used for all metadata in the database, as well as the default for all string columns within the database. Users can choose a different collation for any particular column by using the `COLLATE` clause.  
  
### Example 1
 For example, if we were working in Beijing, we might use a Chinese collation:  
  
```sql  
ALTER DATABASE MyDB COLLATE Chinese_Simplified_Pinyin_100_CI_AS;  
```  
  
 Now if we create a column, its default collation will be this Chinese collation, but we can choose another one if we want:  
  
```sql  
CREATE TABLE MyTable  
      (mycolumn1 nvarchar,  
      mycolumn2 nvarchar COLLATE Frisian_100_CS_AS);  
GO  
SELECT name, collation_name  
FROM sys.columns  
WHERE name LIKE 'mycolumn%' ;  
GO  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
```
name            collation_name  
--------------- ----------------------------------  
mycolumn1       Chinese_Simplified_Pinyin_100_CI_AS  
mycolumn2       Frisian_100_CS_AS  
```  
  
 This appears relatively simple, but several problems arise. Because the collation for a column is dependent on the database in which the table is created, problems arise with the use of temporary tables which are stored in `tempdb`. The collation of `tempdb` usually matches the collation for the instance, which does not have to match the database collation.  
  
### Example 2
 For example, consider the (Chinese) database above when used on an instance with a **Latin1_General** collation:  
  
```sql  
CREATE TABLE T1 (T1_txt nvarchar(max)) ;  
GO  
CREATE TABLE #T2 (T2_txt nvarchar(max)) ;  
GO  
```  
  
 At first glance, these two tables look like they have the same schema, but since the collations of the databases differ, the values are actually incompatible:  
  
```sql
SELECT T1_txt, T2_txt  
FROM T1   
JOIN #T2   
    ON T1.T1_txt = #T2.T2_txt  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
 Msg 468, Level 16, State 9, Line 2  
  
 Cannot resolve the collation conflict between "Latin1_General_100_CI_AS_KS_WS_SC" and "Chinese_Simplified_Pinyin_100_CI_AS" in the equal to operation.  
  
 We can fix this by explicitly collating the temporary table. [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] makes this somewhat easier by providing the `DATABASE_DEFAULT` keyword for the `COLLATE` clause.  
  
```sql  
CREATE TABLE T1 (T1_txt nvarchar(max)) ;  
GO  
CREATE TABLE #T2 (T2_txt nvarchar(max) COLLATE DATABASE_DEFAULT);  
GO  
SELECT T1_txt, T2_txt  
FROM T1   
JOIN #T2   
    ON T1.T1_txt = #T2.T2_txt ;  
```  
  
 This now runs without error.  
  
 We can also see collation-dependent behavior with variables. Consider the following function:  
  
```sql
CREATE FUNCTION f(@x INT) RETURNS INT  
AS BEGIN   
      DECLARE @I INT = 1  
      DECLARE @İ INT = 2  
      RETURN @x * @i  
END;  
```  
  
 This is a rather peculiar function. In a case-sensitive collation, the `@i` in the return clause cannot bind to either `@I` or `@İ`. In a case-insensitive Latin1_General collation, `@i` binds to `@I`, and the function returns `1`. But in a case-insensitive Turkish collation, `@i` binds to `@İ`, and the function returns 2. This can wreak havoc on a database that moves between instances with different collations.  
  
## Contained Databases
 Since a design objective of contained databases is to make them self-contained, the dependence on the instance and `tempdb` collations must be severed. To do this, contained databases introduce the concept of the catalog collation. The catalog collation is used for system metadata and transient objects. Details are provided below.  
  
 In a contained database, the catalog collation is **Latin1_General_100_CI_AS_WS_KS_SC**. This collation is the same for all contained databases on all instances of [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] and cannot be changed.  
  
 The database collation is retained, but is only used as the default collation for user data. By default, the database collation is equal to the `model` database collation, but can be changed by the user through a `CREATE` or `ALTER DATABASE` command as with non-contained databases.  
  
 A new keyword, `CATALOG_DEFAULT`, is available in the `COLLATE` clause. This is used as a shortcut to the current collation of metadata in both contained and non-contained databases. That is, in a non-contained database, `CATALOG_DEFAULT` will return the current database collation, since metadata is collated in the database collation. In a contained database, these two values may be different, since the user can change the database collation so that it does not match the catalog collation.  
  
 The behavior of various objects in both non-contained and contained databases is summarized in this table:  
  
|Item|Non-Contained Database|Contained Database|  
|-|-|-|  
|User Data (default)|DATABASE_DEFAULT|DATABASE_DEFAULT|  
|Temp Data (default)|`tempdb` Collation|DATABASE_DEFAULT|  
|Metadata|DATABASE_DEFAULT / CATALOG_DEFAULT|CATALOG_DEFAULT|  
|Temporary Metadata|`tempdb` Collation|CATALOG_DEFAULT|  
|Variables|Instance Collation|CATALOG_DEFAULT|  
|Goto Labels|Instance Collation|CATALOG_DEFAULT|  
|Cursor Names|Instance Collation|CATALOG_DEFAULT|  
  
 In the temp table example previously described, we can see that this collation behavior eliminates the need for an explicit `COLLATE` clause in most temp table uses. In a contained database, this code now runs without error, even if the database and instance collations differ:  
  
```sql  
CREATE TABLE T1 (T1_txt nvarchar(max)) ;  
GO  
CREATE TABLE #T2 (T2_txt nvarchar(max));  
GO  
SELECT T1_txt, T2_txt  
FROM T1   
JOIN #T2   
    ON T1.T1_txt = #T2.T2_txt ;  
```  
  
 This works because both `T1_txt` and `T2_txt` are collated in the database collation of the contained database.  
  
## <a id="#crossing-between-contained-and-uncontained-contexts"></a> Cross between contained and un-contained contexts

 As long as a session in a contained database remains contained, it must remain within the database to which it connected. In this case the behavior is very straightforward. But if a session crosses between contained and non-contained contexts, the behavior becomes more complex, since the two sets of rules must be bridged. This can happen in a partially-contained database, since a user may `USE` to another database. In this case, the difference in collation rules is handled by the following principle.  
  
-   The collation behavior for a batch is determined by the database in which the batch begins.  
  
 Note that this decision is made before any commands are issued, including an initial `USE`. That is, if a batch begins in a contained database, but the first command is `USE` to a non-contained database, the contained collation behavior will still be used for the batch. Given this, a reference to a variable, for example, may have multiple possible outcomes:  
  
-   The reference may find exactly one match. In this case, the reference will work without error.  
  
-   The reference may not find a match in the current collation where there was one before. This will raise an error indicating that the variable does not exist, even though it was apparently created.  
  
-   The reference may find multiple matches that were originally distinct. This will also raise an error.  
  
 We'll illustrate this with a few examples. For these we assume there is a partially-contained database named `MyCDB` with its database collation set to the default collation, **Latin1_General_100_CI_AS_WS_KS_SC**. We assume that the instance collation is **Latin1_General_100_CS_AS_WS_KS_SC**. The two collations differ only in case sensitivity.  
  
### Example 1
 The following example illustrates the case where the reference finds exactly one match.  
  
```sql
USE MyCDB;  
GO  
  
CREATE TABLE #a(x int);  
INSERT INTO #a VALUES(1);  
GO  
  
USE master;  
GO  
  
SELECT * FROM #a;  
GO  
  
Results:  
  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
```  
x  
-----------  
1  
```  
  
 In this case, the identified #a binds in both the case-insensitive catalog collation and the case-sensitive instance collation, and the code works.  
  
### Example 2

 The following example illustrates the case where the reference does not find a match in the current collation where there was one before.  
  
```sql
USE MyCDB;  
GO  
  
CREATE TABLE #a(x int);  
INSERT INTO #A VALUES(1);  
GO  
```  
  
 Here, the `#A` binds to `#a` in the case-insensitive default collation, and the insert works,  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
```  
(1 row(s) affected)  
```  
  
 But if we continue the script...  
  
```sql
USE master;  
GO  
  
SELECT * FROM #A;  
GO  
```  
  
 We get an error trying to bind to `#A` in the case-sensitive instance collation;  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
 Msg 208, Level 16, State 0, Line 2  
  
 Invalid object name '#A'.  
  
### Example 3

 The following example illustrates the case where the reference finds multiple matches that were originally distinct. First, we start in `tempdb` (which has the same case-sensitive collation as our instance) and execute the following statements.  
  
```sql
USE tempdb;  
GO  
  
CREATE TABLE #a(x int);  
GO  
CREATE TABLE #A(x int);  
GO  
INSERT INTO #a VALUES(1);  
GO  
INSERT INTO #A VALUES(2);  
GO  
```  
  
 This succeeds, since the tables are distinct in this collation:  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
  
```  
(1 row(s) affected)  
(1 row(s) affected)  
```  
  
 If we move into our contained database, however, we find that we can no longer bind to these tables.  
  
```sql
USE MyCDB;  
GO  
SELECT * FROM #a;  
GO  
```  
  
 [!INCLUDE[ssResult](../../includes/ssresult-md.md)]  
 
``` 
 Msg 12800, Level 16, State 1, Line 2  
  
 The reference to temp table name #a is ambiguous and cannot be resolved. Possible candidates are #a and #A.  
```
  
## Next steps

- [Contained Databases](../../relational-databases/databases/contained-databases.md)
