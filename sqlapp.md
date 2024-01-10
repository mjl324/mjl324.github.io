## SQL Application

**Project description:** Using three different data sources with differing and overlapping attributes, I wanted to unify these schemas for a complete SQL application that demonstrated the environmental impacts of the tech industry, especially in light of the explosion in AI research. 

### 1. Data Sources

The USA Tech Companies Stats dataset from Kaggle[https://www.kaggle.com/datasets/lamiatabassum/top-50-us-tech-companies-2022-2023-dataset/data] provides data about the top 50 tech companies in the US for 2022-2023
Emissions by State
Next, I used a dataset on per capita energy-related carbon dioxide emissions by state from 1970-2021 US Energy Information Administration (EIA)
Per capita energy-related carbon dioxide emissions by state from 1970-2021.
NASDAQ-100 Stock Price Data
Source: Kaggle
Data about stock prices of all NASDAQ-100 index stocks from 2010-2021.


### 2. 3rd Normal Form

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

### 3. Loading the Tables

```sql
CREATE TABLE company_info (
  name VARCHAR(40),
  revenue_22_23_e9 VARCHAR(100),
  market_cap_e12 DOUBLE(4,3),
  emp_num INTEGER,
  founded YEAR,
  incomeTax_22_23_USD_e9 DOUBLE(5,3),
  sector VARCHAR(30),
  state VARCHAR(15),
  PRIMARY KEY (name)
  );
LOAD DATA LOCAL INFILE '/Users/meredithlou/Downloads/companies.csv'
  INTO TABLE company_info
  FIELDS TERMINATED BY ','
  IGNORE 1 ROWS;

CREATE TABLE stock_change (
  code VARCHAR(5),
  year YEAR,
  high FLOAT,
  low FLOAT,
  change_in_close FLOAT,
  PRIMARY KEY (code, year)
  );
LOAD DATA LOCAL INFILE '/Users/meredithlou/Downloads/stocks.csv'
  INTO TABLE stock_change
  FIELDS TERMINATED BY ','
  IGNORE 1 ROWS;

CREATE TABLE ticker_symbols (
  name VARCHAR(40),
  code VARCHAR(5),
  FOREIGN KEY (name) references company_info (name),
  FOREIGN KEY (code) references stock_change (code),
  PRIMARY KEY (name)
  );
LOAD DATA LOCAL INFILE '/Users/meredithlou/Downloads/name_code_map.csv'
  INTO TABLE ticker_symbols
  FIELDS TERMINATED BY ','
  IGNORE 1 ROWS;

CREATE TABLE emissions (
  state VARCHAR(15),
  year YEAR,
  emissions_per_cap DOUBLE(3,1),
  PRIMARY KEY (state, year)
  );
LOAD DATA LOCAL INFILE '/Users/meredithlou/Downloads/emissions.csv'
  INTO TABLE emissions
  FIELDS TERMINATED BY ','
  IGNORE 1 ROWS;
```

### 4. Output

```sql
#list the top 10 companies in descending order of positive change in stock value from 2015
SELECT a.name, b.change_in_close
	FROM ticker_symbols a, stock_change b 
	WHERE a.code = b.code
	AND year = 2015
	ORDER BY b.change_in_close DESC
	LIMIT 10;

#list the number of companies from the database headquartered in each state
SELECT state, COUNT(state)
	FROM company_info
	GROUP BY state;

#list the 10 states with the highest emissions in 2021
SELECT state, emissions_per_cap
	FROM emissions
	WHERE year = 2021
	ORDER BY emissions_per_cap DESC
	LIMIT 10;

#list the total revenue_22_23_e9 from the tech companies in our database by sector in 2022-2023
SELECT sector, SUM(revenue_22_23_e9)
	FROM company_info
	GROUP BY sector;

#What company had the biggest value loss in its stock value in 2017?
SELECT code, change_in_close
	FROM stock_change
	WHERE year = 2017
	ORDER BY 2 DESC
	LIMIT 1;

#Which companies had a change in their stock value of between -10 and 10 in 2015?
SELECT a.name
	FROM ticker_symbols a, stock_change b
	WHERE a.code = b.code 
	AND year = 2015
	AND b.change_in_close BETWEEN -10 and 10;

#During which years did Apple have an increase in stock value of at least $100 AND the emissions in the state it’s headquartered were greater than the average emissions?
SELECT DISTINCT a.year
	FROM stock_change a, company_info b, emissions c, ticker_symbols d
	WHERE a.change_in_close >= 100 AND 
	c.state = b.state AND
	b.name = d.name AND 
d.code = 'aapl' AND
d.code = a.code AND 
c.emissions_per_cap > (SELECT AVG(c.emissions_per_cap) 
FROM emissions c, company_info b, stock_change a, ticker_symbols d 
WHERE c.state = b.state AND
	b.name = d.name AND 
d.code = 'aapl' AND
d.code = a.code) ;
```
