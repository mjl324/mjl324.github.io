## DBMS Implementation

**Project description:** This simplified, single-user relational database management system implementation includes a SQL parser, an indexing structure, a query optimizer, and an execution engine using Python.

For more details see [code here](https://gitfront.io/r/mjl324/Utg1tydbeCrX/dbms/).

### 1. Structure

I implemented the database system using a fully indexed structure. The underlying data structure was a hash table in the form of a Python dictionary. 
In a given relation, I stored a global dictionary of {table name:Table object} pairs. Each Table object consisted of a central dictionary, which had a nested sub-dictionary assigned to the name of each column in the table. For the sub-dictionary corresponding to the primary key column, the sub-dictionary value for each primary key is the collection of {column name:column value} pairs in that row for all non-key columns in the table. For the sub-dictionaries corresponding to the non-key columns, the sub-dictionary keys were the unique values of the given column, and the sub-dictionary values were lists of all the primary key values which had the given sub-dictionary key in its row.
I chose to implement this structure to emphasize query run time efficiency over space efficiency. My decision to default to both a fully indexed structure and to use Python dictionaries made our implementation very space inefficient. This is because I store every entry of a given table, including duplicates, in the primary key sub-dictionary, and then store all the unique values of every column again in the non-key sub-dictionaries. 
Despite this space inefficiency, the structure allowed for much faster query computation time, as all columns were indexed. If you wanted to subset your data for a certain column, it only needs to condition the unique values of the column and then get the corresponding lists of primary keys—rather than iterating over every row. Given that the functionality for selection, deletion, and updating all involved first getting a list of conditioned values, the structure made all of these functionalities very efficient.

<img src="images/table structure.jpeg?raw=true"/>
**Figure 1:** A diagram representing the dictionary structure of the system.

### 2. SQL Parsing

The system handles the following data types using the respective syntax:
  - Integers: INT
  - Float: FLOAT
  - String: VARCHAR(size)

The DBMS takes regular SQL syntax with the exception of equi-join language, which deviates from SQL. A join clause in a sample input line of my DBMS would be done by entering the following:

```sql
SELECT * FROM df1 as a, df2 as b JOIN ON a.x1 = b.z1
```

The parser for data definition language is able to handle CREATE TABLE and DROP TABLE, where deleting a table will always drop on cascade. I did not include functionality for CREATE/DROP INDEX because the architecture of the DBMS as a dictionary automatically creates an index for every column. 

For data manipulation language, INSERT, DELETE, and UPDATE are parsed similar to SQL. The INSERT operator contains error handling that will flag when an entry with the wrong data type is being inserted. It also handles when a duplicate entry is being inserted in the primary key and when an entry that does not exist in the primary key is being inserted into a foreign attribute that references the primary key. 
The DELETE operator always will always delete on cascade, meaning that it will delete all instances of a value in all columns that reference the attribute to be deleted.
The UPDATE operator performs a standard value update given a condition. As with insertion, the new value to which entries are to be changed must be able to be converted to the appropriate data type for the column in which the update occurs. Additionally, the user is not able to update the primary key column as that is not allowed by SQL.

For SELECT statements, the DBMS supports the use of table aliases (ex. select a.col1 from tbl1 as a, select a.col1 from tbl1 a, or select col1 from tbl1). For conditions with the WHERE clause, available arithmetic conditioning operators are <, >, <=, >=,  ==, !=. The DBMS also allows for conditioning of strings, using the operators LIKE and NOT LIKE, as well as lists with the use of IN and NOT IN.
As discussed briefly in the description of the overall architecture, each condition of our SELECT statement will subset the sub-dictionary keys for the appropriate column and get the list of primary-key values that meet the condition. With a list for each column, the lists are simply combined with the appropriate logic (AND or OR) and return the final list of primary-keys that fit the SELECT statement.

The DBMS allows for variable-length conjunctive or disjunctive queries on the same relation. This means that there is no limitation on the number of conditions, as long as the logic for the whole statement is either always AND or always OR. For commands with multiple conjunctive expressions, we send each sublist to the AND optimizer, which then returns a nested list of keys that fulfill each condition, sorted in order of low selectivity to high. The unique indexing structure supports quicker conditioning by using just lists of keys, making it O(1) to grab each list rather than iterating through each one. For commands with multiple disjunctive expressions, the system sends each sublist to the OR optimizer, which then returns only one list of unique keys that fulfill at least one of the conditions.

The DBMS supports four aggregation operators; MIN, which finds the minimum value in a certain attribute, MAX, which finds the maximum value of an attribute, SUM, which sums the values in an attribute, and AVG, which calculates the average value of an attribute. The chosen attribute can be modified before the aggregation operator is applied. However, because there is no GROUP BY functionality, if an aggregation operator is applied, the only attribute which may be selected is the attribute to which the operator is applied. 

### 3. Joining

The DBMS had functionality to join across two databases through an equi-join. This equi-join was an inner join which can join across at most two databases, across only one attribute per database (i.e. JOIN ON a.name = b.Letter). This join could be performed over key or non-key attributes.
There were multiple optimization tactics used during the joining process. Firstly, I used a query tree optimization by always performing any necessary subsets prior to joining across relations (ex. a.name == “aaa” will always be performed before a.name = b.Letter). While this may occasionally result in a less efficient sequence, this will usually result in faster code and also does not require any time to compute specific values. Secondly, I included two separate joining strategies in separate functions (merge-scan and nested loop joining). To determine which strategy was used, I performed a cost-based optimization by determining the approximate amount of time required to perform either a nested loop or merge-scan join, and then used the strategy which would take less time. The input for the join function was always two lists of keys from each set to be joined; for the first set of length n, and the second set of length m, the merge-scan cost was calculated to be nlogn + mlogm + n + m, and the nested loop cost was calculated to be nm. Although these functions must be calculated every time join is called, they will almost always result in the quickest path being chosen.


### 4. Output

Once any necessary joins and subsetting have been performed, we now have a final list of primary keys associated with the data we wish to output (assuming the input was a SELECT statement; if not, these steps are skipped)! Firstly, the system checks to see if any aggregation operators are called; if so, the necessary operations are performed and the relevant operator is applied to the desired attribute and outputted. Otherwise, it finds the attributes that must be outputted, and output the values in this column associated with the final list of primary keys. The final output includes the amount of time to execute the statement, and a table of the desired output (if the statement is a query).

The below queries use tables from the example data to demonstrate the various functionalities of my DBMS:

First, I load df1 and df2 into tables and create a df3. 

```sql
CREATE TABLE df1 (Letter VARCHAR(3), Number INT, Color VARCHAR(6), PRIMARY KEY(Letter));
LOAD DATA INFILE  'data/df1.csv' INTO TABLE df1 IGNORE 1 ROWS;
CREATE TABLE df2 (name VARCHAR(3), decimal FLOAT, state VARCHAR(10), year INT, FOREIGN KEY (name) REFERENCES df1(Letter), PRIMARY KEY(name));
LOAD DATA INFILE 'data/df2.csv' INTO TABLE df2 IGNORE 1 ROWS;
CREATE TABLE df3 (name VARCHAR(3), color VARCHAR(6), PRIMARY KEY(name));
INSERT INTO df3 (name,color) VALUES (aab,Red);
INSERT INTO df3 (name,color) VALUES (aad,Red);
INSERT INTO df3 (name,color) VALUES (aac,Orange);
```
## Query 2 ########## (Aggregation Operators)
SELECT AVG(Number) from df1 
WHERE Number < 10;

### Query 3 ########## (Join Over Key, OR)
SELECT * FROM df1 a, df2 b 
JOIN ON a.Letter = b.name 
WHERE b.year == 2023 <img width="418" alt="Screenshot 2024-01-10 at 12 51 54 PM" src="https://github.com/mjl324/mjl324.github.io/assets/98557577/6d15e965-80cd-4cb6-b4b5-83c86ce25d9a">

OR b.decimal < 0.1;

### Query 4 ########## (Join Over Non-Key, AND)
SELECT b.name, a.Letter, a.Color, a.Number FROM df1 a, df3 b 
JOIN ON a.Color = b.color 
WHERE a.Color == 'Red'
AND a.Number < 3;

### Query 5 ########## (LIKE, Arithmetic, IN, Multiple ORs)
SELECT * FROM df2 
WHERE name LIKE "aa%" 
OR decimal > 0.9 
OR year IN (2002, 2003, 2004);


### Query 6 ########## (Multiple ANDs, Join Over Key)
SELECT a.Letter, a.Color, b.decimal, b.year FROM df1 a, df2 b 
JOIN ON a.Letter = b.name 
WHERE b.year < 2004 
AND b.decimal < 0.75 
AND b.decimal > 0.5;

Class data:

CREATE TABLE df1 (w1 INT, w2 INT, PRIMARY KEY(w1));
CREATE TABLE df2 (x1 INT, x2 INT, FOREIGN KEY (x1) REFERENCES df1(w1), PRIMARY KEY(x1));
CREATE TABLE df3 (y1 INT, y2 INT, PRIMARY KEY(y1));
CREATE TABLE df4 (z1 INT, z2 INT, FOREIGN KEY (z1) REFERENCES df3(y1), PRIMARY KEY(z1));
LOAD DATA INFILE 'data/rel_i_i_1000' INTO TABLE df1 IGNORE 1 ROWS;
LOAD DATA INFILE 'data/rel_i_1_1000' INTO TABLE df2 IGNORE 1 ROWS;
LOAD DATA INFILE 'data/rel_i_i_10000' INTO TABLE df3 IGNORE 1 ROWS;
LOAD DATA INFILE 'data/rel_i_1_10000' INTO TABLE df4 IGNORE 1 ROWS;

### Outputting Full Tables ##########

SELECT * FROM df1;

SELECT * FROM df2;

SELECT * FROM df3;

SELECT * FROM df4;

### Query 1 ########## (Good Insertion)
INSERT INTO df1 (w1, w2) VALUES (1001,1001);
SELECT * FROM df1 WHERE w1 > 990;

### Query 2 ########## (Duplicate Primary Key)
INSERT INTO df1 (w1, w2) VALUES (1,1);

### Query 3 ########## (Non-Existent Foreign Attribute)
INSERT INTO df2 (x1, x2) VALUES (9999,1);

### Query 4 ########## (Insertion with Wrong Data Type)
INSERT INTO df2 (x1, x2) VALUES (‘hello’,1);


### Query 5 ########## (Good Deletion, Error Handling, Cascading Delete)
DELETE FROM df1 WHERE w1 == 1000;
SELECT * FROM df1 WHERE w1 > 990;

DELETE FROM df1 WHERE w1 == 1000;

SELECT * FROM df2 WHERE x1 > 990;

### Query 6 ########## (Good Update)
UPDATE df1 SET w2 = 8888 WHERE w2 > 995;
SELECT * FROM df1 WHERE w1 > 990;

### Query 7 ########## (Update Key Column)
UPDATE df1 SET w1 = 8888 WHERE w1 > 995;

### Query 8 ########## (Update With Wrong Data Type)
UPDATE df1 SET w2 = “hello” WHERE w2 > 995;

### Query 9 ########## (Cascading Drop)
DROP TABLE df1;

SELECT * FROM df1;

SELECT * FROM df2;

### Query 10 ########## (Nested Loop)

CREATE TABLE df5 (f1 INT, f2 INT, PRIMARY KEY(f1));
INSERT INTO df5 (f1, f2) VALUES (1,1);

SELECT a.f1, b.z1 FROM df5 a, df4 b JOIN ON a.f1 = b.z1;

### Query 11 ########## (Merge Scan)

SELECT a.y1, b.z1 FROM df3 a, df4 b JOIN ON a.y1 = b.z1;
```
