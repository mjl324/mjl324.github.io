## SQL Application

**Project description:** Using three different data sources with differing and overlapping attributes, I wanted to unify these schemas for a complete SQL application that demonstrated the environmental impacts of the tech industry, especially in light of the explosion in AI research. 

### 1. Data Sources

The USA Tech Companies Stats dataset from [Kaggle](https://www.kaggle.com/datasets/lamiatabassum/top-50-us-tech-companies-2022-2023-dataset/data) provides data about the top 50 tech companies in the US for 2022-2023

Next, I used a dataset on per capita energy-related carbon dioxide emissions by state from 1970-2021, taken from the US Energy Information Administration (EIA) [website](https://www.eia.gov/environment/emissions/state/).

Finally, I also looked at [data](https://www.kaggle.com/datasets/kalilurrahman/nasdaq100-stock-price-data) on the stock prices of all NASDAQ-100 index stocks from 2010-2021.


### 2. 3rd Normal Form

With the [cleaned data](), the schema is in second normal form. It is not 3NF because there is a transitive dependency in the Tech Company Stats dataset on stock code, which is tied to the field Company Name.

```javascript
SELECT * FROM df1 as a, df2 as b JOIN ON a.x1 = b.z1
```

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
