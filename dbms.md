## DBMS Implementation

**Project description:** This simplified, single-user relational database management system implementation includes a SQL parser, an indexing structure, a query optimizer, and an execution engine using Python.

For more details see [code here](https://guides.github.com/features/mastering-markdown/).

### 1. Structure

I implemented the database system using a fully indexed structure. The underlying data structure was a hash table in the form of a Python dictionary. 
In a given relation, I stored a global dictionary of {table name:Table object} pairs. Each Table object consisted of a central dictionary, which had a nested sub-dictionary assigned to the name of each column in the table. For the sub-dictionary corresponding to the primary key column, the sub-dictionary value for each primary key is the collection of {column name:column value} pairs in that row for all non-key columns in the table. For the sub-dictionaries corresponding to the non-key columns, the sub-dictionary keys were the unique values of the given column, and the sub-dictionary values were lists of all the primary key values which had the given sub-dictionary key in its row.
I chose to implement this structure to emphasize query run time efficiency over space efficiency. My decision to default to both a fully indexed structure and to use Python dictionaries made our implementation very space inefficient. This is because I store every entry of a given table, including duplicates, in the primary key sub-dictionary, and then store all the unique values of every column again in the non-key sub-dictionaries. 
Despite this space inefficiency, the structure allowed for much faster query computation time, as all columns were indexed. If you wanted to subset your data for a certain column, it only needs to condition the unique values of the column and then get the corresponding lists of primary keys—rather than iterating over every row. Given that the functionality for selection, deletion, and updating all involved first getting a list of conditioned values, the structure made all of these functionalities very efficient.

<img src="images/tablestructure.jpeg?raw=true"/>
**Figure 1:** A diagram representing the dictionary structure of the system.

### 2. SQL Parsing

The system handles the following data types using the respective syntax:
  - Integers: INT
  - Float: FLOAT
  - String: VARCHAR(size)

The DBMS takes regular SQL syntax with the exception of equi-join language, which deviates from SQL. A join clause in a sample input line of my DBMS would be done by entering the following:

```javascript
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
