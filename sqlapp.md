## SQL Application

**Project description:** Using three different data sources with differing and overlapping attributes, I wanted to unify these schemas for a complete SQL application that demonstrated the environmental impacts of the tech industry, especially in light of the explosion in AI research. 

For more details see [code here](https://gitfront.io/r/mjl324/KKXHGra3iS89/sqlapp/).

### 1. Data Sources

The USA Tech Companies Stats dataset from [Kaggle](https://www.kaggle.com/datasets/lamiatabassum/top-50-us-tech-companies-2022-2023-dataset/data) provides data about the top 50 tech companies in the US for 2022-2023

Next, I used a dataset on per capita energy-related carbon dioxide emissions by state from 1970-2021, taken from the US Energy Information Administration (EIA) [website](https://www.eia.gov/environment/emissions/state/).

Finally, I also looked at [data](https://www.kaggle.com/datasets/kalilurrahman/nasdaq100-stock-price-data) on the stock prices of all NASDAQ-100 index stocks from 2010-2021.


### 2. 3rd Normal Form

With the [cleaned data](https://gitfront.io/r/mjl324/KKXHGra3iS89/sqlapp/blob/datacleaning.py), the schema is in second normal form. It is not 3NF because there is a transitive dependency in the Tech Company Stats dataset on stock code, which is tied to the field Company Name. 
<img width="943" alt="Screenshot 2024-01-10 at 2 12 42 PM" src="https://github.com/mjl324/mjl324.github.io/assets/98557577/2a86f89a-f909-40bc-a273-9672372d0ab0">

In order to make the relations in third normal form, I had to [adjust](https://gitfront.io/r/mjl324/KKXHGra3iS89/sqlapp/blob/normalform.py) the data further.
<img width="934" alt="Screenshot 2024-01-10 at 2 13 05 PM" src="https://github.com/mjl324/mjl324.github.io/assets/98557577/56c51010-81c9-4290-9351-1d8d3efe5ea1">


### 3. Loading the Tables

Now that the data is in 3NF, we can now load the tables!

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

With the tables loaded, we are now ready to perform queries on our database.

This database can provide information on tech company performance, changes in stock value over time/presently, size of company, revenue, andmore!
Additionally, there are metrics available to see state carbon impact and emissions per capita over time. We can make easy comparisons between performance of company and emissions in the state they are headquartered in.


If we are interested in the top 10 companies in descending order of positive change in stock value from 2015:
```sql
SELECT a.name, b.change_in_close
	FROM ticker_symbols a, stock_change b 
	WHERE a.code = b.code
	AND year = 2015
	ORDER BY b.change_in_close DESC
	LIMIT 10;
```

If we are looking for the number of companies from the database headquartered in each state:

```sql
SELECT state, COUNT(state)
	FROM company_info
	GROUP BY state;
```

The 10 states with the highest emissions in 2021:

```sql 
SELECT state, emissions_per_cap
	FROM emissions
	WHERE year = 2021
	ORDER BY emissions_per_cap DESC
	LIMIT 10;
```

Total revenue from 2022-2023 from the tech companies by sector:

```sql
SELECT sector, SUM(revenue_22_23_e9)
	FROM company_info
	GROUP BY sector;
```

The company that had the biggest value loss in its stock value in 2017:
```sql
SELECT code, change_in_close
	FROM stock_change
	WHERE year = 2017
	ORDER BY 2 DESC
	LIMIT 1;
```

The companies that had a change in their stock value of between -10 and 10 in 2015:
```sql
SELECT a.name
	FROM ticker_symbols a, stock_change b
	WHERE a.code = b.code 
	AND year = 2015
	AND b.change_in_close BETWEEN -10 and 10;
```

The years that Apple saw both an increase in stock value of at least $100 and higher than average emissions in the state in which it is headquarted:

```sql
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
	d.code = a.code);
```
