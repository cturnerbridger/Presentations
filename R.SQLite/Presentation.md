## Agenda

* Brief overview of databases
* DML and DDL
* Doing CRUD
* Some DDL
* Some best practices

# Databases & drivers
## What are databases?
> A database is a collection of data

Practically, a database is a **persistent** repository of data that has a programmatic **interface** for management and access of the data.

## How is a relational database structured?
```{r, echo=FALSE, message=FALSE}
library(data.table)
library(ggplot2)
ratio<-2/3
dblayout<-rbind(
  data.table(level=4,colour='DB',name='Database', start=0      ,end=1),
  data.table(level=3,colour='S', name='Schema'  , start=0      ,end=ratio),
  data.table(level=3,colour='S', name='...'     , start=ratio  ,end=1),
  data.table(level=2,colour='T', name='Table'   , start=0      ,end=ratio^2),
  data.table(level=2,colour='T', name='...'     , start=ratio^2,end=ratio),
  data.table(level=1,colour='C', name='Key'     , start=0      ,end=ratio^3),
  data.table(level=1,colour='C', name='...'     , start=ratio^3,end=ratio^2)
)

ggplot(dblayout
       ,aes(xmin=level-1,xmax=level,ymin=start,ymax=end, fill=colour))+
  geom_rect(colour="white",size=3)+
  coord_flip()+
  geom_text(aes(x=level-0.5,y=start+.05,label=name)
            ,hjust=0,colour="white",size=13 )+
  theme_void()+theme(legend.position="none") +
 scale_fill_brewer(type = "qual",palette = 7)
```

## Other database objects
- View: stored read query
- Procedure: stored code that performs one or more actions
- Function: stored logic to act on values

## DB drivers
Database drivers interface with the DBMS programmatically

- Native drivers are often the fastest but tend to be more restricted to OS
- Open Database Protocol Client (ODBC) is very common and more portable. Slow
- Java Database Protocol Client (JDBC) is less common but sometimes required. Slower

> You need the right driver for your database  
> You may need to consider 32 or 64-bit

# SQLite vs other databases
## SQLite
SQLite is "the world's most deployed database" as it has a very small footprint, low dependencies, and most SQL typically required.

Good for:

- "toy databases"
- use as a local cache for IoT and offline apps
- unchanging schema
- storage of intermediate results

Not so good for:

- Data Loss (difficult to change / modify tables)
- Security
- Reliability

## Other databases (RDBMS')
Other databases like SQL Server, MySQL, Oracle have a greater footprint than SQLite.

Good for:

- large data volumes
- more complex SQL commands
- changing schema
- central storage

# Working with databases in R

## Recap
You need a database, either:

- a local one, or
- the connection details for a remote one

You need the relevant driver

## Then ...

>- If you're using ODBC drivers you can use `RODBC`
>- If you're using JDBC drivers you can use `RJDBC`
>- If you're using a variety of drivers you can use `DBI`
>- If you're using a native driver then there could be a package e.g. `RSQLite`

## Let's get started
Create or access a local SQLite file
```{r, echo=FALSE, results='hide'}
if(file.exists("local.db")) file.remove("local.db")
rowcheck<-function(tbl="airquality") nrow(dbReadTable(myDB,tbl))
```

```{r, message=FALSE}
library(RSQLite)

# Sample DB
dbListTables(datasetsDb())[1:2]

# Create our own
myDB<-dbConnect(SQLite(),"local.db")
dbListTables(myDB)
```

## Sneak peak!
```{r}
qResult<-dbWriteTable(
          myDB,"airquality",airquality)
aq     <-dbGetQuery(
          myDB,"SELECT * FROM airquality
                      WHERE ozone IS NULL")
head(aq)
```

## Exercise 1
* Create a database on your machine

# D[X]L

## Data Definition Lanuage (DDL)
DDL describes SQL keywords that allow you to manage the structures data is held in. Note: owing to likely database restrictions you will not be able to do much, if any, of this.

Examples include:

- `CREATE TABLE`
- `ALTER TABLE`
- `DROP TABLE`
- `CREATE INDEX`

Point on 'Create Index': it is worth understanding and taking time to understand indexing in data.tables to make your queries faster.

## Data Manipulation Language (DML)
DML describes SQL keywords that allow you to manage the data itself. CRUD covers the main operations. This is what you will likely be doing.

## What’s CRUD?

>- **C**reate = `INSERT`
>- **R**ead = `SELECT`
>- **U**pdate = `UPDATE`
>- **D**elete = `DELETE`

>- Upsert = `MERGE`

# Performing CRUD actions

## INSERT | About
* INSERT statements take a set of values and map how they should enter into the table
* An INSERT can either take a SQL result set or some predefined VALUEs


## INSERT | Example
```sql
INSERT INTO airquality 
SELECT * FROM airquality
WHERE ozone IS NULL
```

```sql
INSERT INTO airquality (ozone, month, temp)
VALUES (87, 6, 70)
```
```{r,echo=FALSE,results='hide'}
dbRemoveTable(myDB,"airquality")
```

## INSERT | In R
Option 1: `CREATE` and `INSERT` using `dbWriteTable`
```{r}
qResult<-dbWriteTable(
          myDB,"airquality",airquality)
rowcheck()
```

## INSERT | In R
Option 2: `INSERT` using `dbWriteTable` with `append=TRUE`
```{r}
newdata<-airquality[is.na(airquality$Ozone),]
qResult<-dbWriteTable(
          myDB,"airquality",
          newdata,
          append=TRUE)
rowcheck()
```

## INSERT | In R
Option 3: `INSERT` using `dbGetPreparedQuery`

NB. This looks like a lot but this **parameterises** the query (more on that later!)
```{r}
sql<-paste0("INSERT INTO airquality VALUES ("
            ,paste(rep("?",ncol(newdata))
                   ,collapse = ",")
            ,")")

dbGetPreparedQuery(
          myDB,sql,
          bind.data = newdata)

rowcheck()
```

## Exercise 2
* Add the iris data to your database

## SELECT | About
* SELECT is the statement that retrieves data from tables
* Allows us to define the columns we want by name e.g. `ozone, wind`
* Can simply return all columns with \*
* FROM dictates what table to retrieve data from
* We can also add additional elements:
    + JOINs
    + WHERE 
    + GROUP BY
    + HAVING
    + ORDER BY

## SELECT | Example
```sql
SELECT * FROM airqualilty;
SELECT ozone, wind, month FROM airquality;
```
## SELECT | In R
Option 1: `SELECT` data using `dbReadTable`
```{r}
aqA <-dbReadTable(myDB,"airquality")
aqG <-dbReadTable(myDB,"airquality"
                  ,select.cols="ozone, wind")

ncol(aqA)
ncol(aqG)
```

## SELECT | In R
Option 2: `SELECT` data using `dbGetQuery`
```{r}
aqSQL<-dbGetQuery(myDB,"SELECT ozone, wind 
                  FROM airquality")
identical(aqG,aqSQL)
```

## Exercise 3
* Get the sepal length and width columns from your iris table - use dbGetQuery

## UPDATE | About
The UPDATE allows you to change values of a records within a table

* Use a WHERE to restrict the UPDATE to only rows you want to amend. Even if the WHERE is [1=1], the reason for this is that it builds in an important habbit.
* * side point: STAR Schema: used to preserve granularity when you collect data into one table. For example, the NULL return may be set as different negitive numbers for different reasons (i.e. no name, no postcode, no credit card...)

## UPDATE | Example
```sql
UPDATE airquality
SET ozone = -99
WHERE ozone IS NULL
```

## UPDATE | In R
```{r}
dbGetQuery(myDB
           ,"UPDATE airquality
            SET wind=wind*1.1
            WHERE day=1")
```

## Exercise 4
You firmly believe in English spellings of words like colour, update *versicolor* to *versicolour*

## DELETE | About
DELETE allows you to remove records from a table

* Should always use a WHERE to prevent the deletion of all rows
* DELETE retains any incremental ID max number. Use TRUNCATE to delete everything and reset the ID col
* Logical deletion is much preferred i.e. add an IsDeleted column

## DELETE | Example
```sql
DELETE FROM airquality
WHERE ozone IS NULL
```

## DELETE | In R
```{r}
rowcheck()
dbGetQuery(myDB
           ,"DELETE FROM airquality
            WHERE month<=4")
rowcheck()
```

## Exercise 5
Remove any iris records where the sepal length is less than 4.5

# Writing effective SELECT statements
## JOINs | About
JOINs relate two tables to each other. Different types of JOIN perform different matches.

* INNER - strict matching, so only records that match the criteria are reurned
* LEFT - less strict, returns everything from our LHS table and includes data that matched criteria
    + RIGHT does this for RHS, you can always rewrite a RIGHT to be a LEFT so do so! (because SQL reads "left to right")
* FULL - not strict, returns everything from the LHS and RHS tables even if no match
* CROSS - no criteria, returns the row combinations of LHS & RHS tables

## JOINS | Some setup
```{r}
monthTbl<-data.frame(month=1:10)
monthTbl$value<-monthTbl$month*rnorm(10)
dbWriteTable(myDB,"months",monthTbl[-5,])
rowcheck("months")
```

## JOINs | Example | INNER
```sql
SELECT * 
FROM airquality a
INNER JOIN months m
  ON m.month=a.month
```

## JOINs | Example | LEFT
```sql
SELECT * 
FROM airquality a
LEFT JOIN months m
  ON m.month=a.month
```

## JOINs | Example | FULL
```sql
SELECT * 
FROM airquality a
FULL JOIN months m
  ON m.month=a.month
```

## JOINs | Example | CROSS
```sql
SELECT * 
FROM (SELECT DISTINCT month 
      FROM airquality) a
CROSS JOIN months m
```

## JOINs | In R
In R, use JOINs as part of the SELECT statement for `dbGetQuery`
```{r}
aqJOIN<-dbGetQuery(myDB
                  ,"SELECT a.ozone, a.wind , m.month
                  FROM airquality a
                  INNER JOIN months m 
                    ON a.month=m.month")
rowcheck()
nrow(aqJOIN)
```

## Exercise 6
Create a data.frame that holds some species names and some info about it

* Add an extra row for the species "unknown"
* Remove the Virginica species

## Exercise 7
Write the data.frame to your database

## Exercise 8
Get summaries by month for each type of JOIN and identify the differences

## WHERE | About
The WHERE clause is where filtering of rows happens. 

* This can be performed in a SELECT, UPDATE, or DELETE statement.
* A WHERE should always resolve to a TRUE/FALSE for every row
* Missings aka NULLs should be taken into account
* Standard comparison operators apply
* Build composite conditions with OR, AND, NOT
* Other predicates: IS [NOT] NULL, LIKE, IN

## WHERE | Example
```sql
SELECT * 
FROM airquality
WHERE ozone IS NULL
  AND (month<=5 OR month=8)
  AND NOT (wind IS NULL)
```

## WHERE | In R
```{r}
aqWHERE<-dbGetQuery(myDB
                  ,"SELECT a.ozone, a.wind , a.month
                  FROM airquality a
                  WHERE a.month>=7")
rowcheck()
nrow(aqWHERE)
```

## Exercise 9
Retrieve all iris specimens with an approximate petal area more than 3 and an approximate sepal area of less than 10

## GROUP BY | About
The GROUP BY allows you to perform aggregations by unique values (or combos where multiple columns specified)

* SQL has standard aggregations e.g. COUNT, SUM, MIN, MAX
* MEAN can vary by RDBMS, MEDIAN is frequently not implemented
* Any columns in your GROUP BY should go in your SELECT
* You should only have GROUP BY columns and aggregations in the SELECT

## GROUP BY | Example
```sql
SELECT month
      , COUNT(*) as volume
      , MIN(ozone) as lowestOzone 
FROM airquality
GROUP BY MONTH
```

## GROUP BY | In R
```{r}
aqGROUP<-dbGetQuery(myDB
                  ,"SELECT a.day, AVG(a.wind) as avgWind
                  FROM airquality a
                  GROUP BY a.day")
head(aqGROUP)
```

## Exercise 10
Calculate the average petal lengths and widths by species

## ORDER BY | About
ORDER BY allows you to sort by columns

* Generally don't bother doing this if bringing data into R, it's excess overhead
* Ascending (ASC) is the default sort order
* Use ASC and DESC to control sort order
* Some RDBMS allow you to control where NULLs appear

## ORDER BY | Example
```sql
SELECT * 
FROM airquality
ORDER BY month DESC, day DESC
```


## ORDER BY | In R
```{r}
aqORDERBY<-dbGetQuery(myDB
                  ,"SELECT a.ozone, a.wind 
                  FROM airquality a
                  ORDER BY wind DESC")
head(aqORDERBY)
```

## Exercise 11
Sort your iris data by approximate petal area, from largest to smallest

# Managing tables
## CREATE tables from R | About
We saw `dbWriteTable` can be used to CREATE a table. We've been doing writing a data.frame but you can also write a CSV

`dbWriteTable` takes the following optional arguments:

* `row.names` - use if you have important row names that you would like made into a column
* `overwrite` - use if you want to DELETE the previous records and INSERT the new ones
* `append` - use if you'd like to INSERT the data to an existing table
* `field.types` - use if you need to force specific data types

NBs:

* The SQL statement `CREATE TABLE` can also be used within `dbGetQuery`
* Text / CSV data can also be used, see`?"dbWriteTable,SQLiteConnection,character,character-method"`

## CREATE tables from R | Example
```sql
CREATE TABLE airquality(
id int AUTO_INCREMENT,
ozone decimal(10,2),
PRIMARY KEY(id)
)
```


## CREATE tables from R | In R
```{r}
aqDEFNONLY<-dbWriteTable(
          myDB,"airquality1",airquality[0,])
rowcheck("airquality1")
```

## Exercise 12
We've played around with our iris data enough, reset the data in the table to the contents of the iris data.frame

## DROPping tables | About
Tables consume space so when they're no longer relevant we can remove them. In SQL this is called dropping the table and is done with the DROP command.

* Use with LOTS of care and trepidation!
* Take a backup where possible just in case

## DROPping tables | Example
```sql
DROP TABLE months
```

## DROPping tables | In R
```{r}
dbRemoveTable(myDB,"airquality")
dbListTables(myDB)
```

## Exercise 13
Drop your iris table

# Best practices
## Audit trails
Data provenance is really important, it is therefore good pracctice to add extra columns that contain:

1. The username of the person INSERTing the data
2. The datetime of when the data was INSERTed
3. The username of the person who last modified the data
4. The datetime of when the data was last modified

TIP: Use `Sys.getenv("USERNAME")` to retrieve your username programatically

## Data types
If your data is a "one-shot" it doesn't matter too much what the data types are but if your data can grow then it's important to account for number growth (or keys) or potential input values e.g. longer names than currently accounted for. Use the `field.types` argument on `dbWriteTable` or use a full CREATE statement.

## Security
When I inserted data earlier I **parameterised** my SQL query. This meant that any "little Bobby Tables" can't break my query.

You should also be careful with storing credentials for your database access. A simple way to do this is store the details in a seperate file that doesn't get source controlled.
![](https://imgs.xkcd.com/comics/exploits_of_a_mom.png)

## Politeness
Always close your connection!
```{r}
dbDisconnect(myDB)
```

# Conclusion
## Wrap-up
Covered a lot of syntax, but not covered everything:

1. Building connections
2. CRUD
3. DDL
4. Best practices

Get this slidedeck: [github.com/mangothecat/mangopresentations](https://github.com/MangoTheCat/MangoPresentations/blob/master/presentations/WorkingWithDatabases.Rmd)

Overview: it is useful to filter data in your SQL queries, then, if possible, pull that data into your memory and manipulate in R using the data.table package.

## Q&A

## Thank you!
