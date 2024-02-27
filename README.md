# DataWarehousing-dvdrental

In this project I'm going to use Git, as well as Azure Cloud (Azure Data Factory, Data Lake Gen 2, Azure Synapse Analytics, Azure Databricks etc.) to build the data pipeline in order to Extract, Transform and Load (ETL) the data from 2021 Olympics in Tokyo.

<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#olympic-analysis-azure">About the project</a>
      <ul>
        <li><a href="#built-with">Built With</a></li>
      </ul>
      <ul>
        <li><a href="#data-source">Data Source</a></li>
      </ul>
      <ul>
        <li><a href="#scenario">Scenario</a></li>
      </ul>
    </li>
    <li>
      <a href="#execution">Execution</a>
      <ul>
        <li><a href="#storage-account-and-resource-group">Storage Account and Resource Group</a></li>
      </ul>
      <ul>
        <li><a href="#data-factory">Data Factory</a></li>
      </ul>
      <ul>
        <li><a href="#databricks">Data Factory</a></li>
      </ul>
      <ul>
        <li><a href="#synapse-analytics">Synapse Analytics</a></li>
      </ul>
    </li>
      <a href="#summary">Summary</a>
    </li>
  </ol>

  
</details>

-----------------------------------------------------------------------------------------

## Built With
1. Data Source - dvdrental dataset (from GitHub)

2. Data Extraction - Git Bash

3. Data Integration - Azure Data Factory

4. Raw Data Store - Azure Data Lake Gen2

5. Data Transformation - Azure DataBricks (PySpark)

6. Transformed Data - Azure Data Lake Gen2

7. Analytics - Azure Synapse Analytics

![](images/icons1withoutpowerbi.png)

-----------------------------------------------------------------------------------------

## Scenario
1. Extract Data using Git Bash
   
2. Copy raw data to Azure Storage (Data Lake Gen2) using Azure Data Factory
   
3. Perform data transformation using Databricks (PySpark)
   
4. Upload transofrmed data to Azure Storage (Data Lake Gen2)
   
5. Understand the data in Azure Synapse Analytics using SQL queries

![](images/dashboard1_nopowerBI.png)

-----------------------------------------------------------------------------------------

## Data Source
The data for this project was provided by postgresqltutorial.com
[postgrestutrial](https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/)

I loaded this data into PostreSQL DataBase using Restore option

ER Diagram of this dataset looks like this:

-----------------------------------------------------------------------------------------

## Execution
### Creating Tables
I wanted to tranform my data into a star schema according to the ER Diagram I created

I started by creating dimDate table 
```sql
DROP TABLE IF EXISTS dimDate;
CREATE TABLE dimDate
(
	date_key integer NOT NULL PRIMARY KEY,
	date date NOT NULL,
	year smallint NOT NULL,
	quarter smallint NOT NULL,
	month smallint NOT NULL, 
	day smallint NOT NULL,
	week smallint NOT NULL,
	is_weekend boolean
);
```

Followed by dimCustomer table 
```sql
DROP TABLE IF EXISTS dimCustomer;
CREATE TABLE dimCustomer
(
	customer_key SERIAL PRIMARY KEY,
	customer_id smallint NOT NULL, 
	first_name varchar(45) NOT NULL,
	last_name varchar(45) NOT NULL,
	email varchar(50),
	address varchar(50) NOT NULL,
	address2 varchar(50),
	district varchar(20) NOT NULL,
	city varchar(50) NOT NULL,
	country varchar(50) NOT NULL,
	postal_code varchar(10),
	phone varchar(20) NOT NULL,
	active smallint NOT NULL,
	create_date timestamp NOT NULL,
	start_date date NOT NULL,
	end_date date NOT NULL
);
```

Next, dimMovie table
```sql
DROP TABLE IF EXISTS dimMovie;
CREATE TABLE dimMovie
(
	movie_key SERIAL PRIMARY KEY,
	film_id smallint NOT NULL, 
	title varchar(255) NOT NULL,
	description text NOT NULL,
	release_year year,
	language varchar(20) NOT NULL,
	original_language varchar(20),
	rental_duration smallint NOT NULL,
	length smallint NOT NULL,
	rating varchar(5) NOT NULL,
	special_features varchar(60) NOT NULL
);
```

Followed by dimStore table 
```sql
DROP TABLE IF EXISTS dimStore;
CREATE TABLE dimStore
(
	store_key SERIAL PRIMARY KEY,
	store_id smallint NOT NULL, 
	address varchar(50) NOT NULL,
	address2 varchar(50),
	district varchar(20) NOT NULL,
	city varchar(50) NOT NULL,
	country varchar(50) NOT NULL,
	postal_code varchar(10),
	manager_first_name varchar(45) NOT NULL,
	manager_last_name varchar(45) NOT NULL,
	start_date date NOT NULL,
	end_date date NOT NULL
);
```

And finally fact table called 'factSales'
```sql
DROP TABLE IF EXISTS factSales;
CREATE TABLE factSales
    (
        sales_key SERIAL PRIMARY KEY,
        date_key integer REFERENCES dimDate (date_key),
        customer_key integer REFERENCES dimCustomer (customer_key),
        movie_key integer REFERENCES dimMovie (movie_key),
        store_key integer REFERENCES dimStore (store_key),
        sales_amount numeric
    );
```

### Inserting the Data

Inserting data from payment table into dimDate table
```sql
INSERT INTO dimDate (date_key,date,year,quarter,month,day,week,is_weekend)
SELECT 
	DISTINCT(TO_CHAR(payment_date :: DATE, 'yyyMMDD')::integer) as date_key,
	date(payment_date) as date,
	EXTRACT(year from payment_date) as year,
	EXTRACT(quarter from payment_date) as quarter,
	EXTRACT(month from payment_date) as month,
	EXTRACT(day from payment_date) as day,
	EXTRACT(week from payment_date) as week,
	CASE WHEN EXTRACT(ISODOW FROM payment_date) IN (6,7) THEN true ELSE false END
FROM payment;
```

Inserting data from customer, address, city and country table into dimCustomer table
```sql
INSERT INTO dimCustomer (customer_key, customer_id, first_name, last_name, email, address, 
                         address2, district, city, country, postal_code, phone, active, 
                         create_date, start_date, end_date)
SELECT  c.customer_id as customer_key,
        c.customer_id,
        c.first_name,
        c.last_name,
        c.email,
        a.address,
        a.address2,
        a.district,
        ci.city,
        co.country,
        postal_code,
        a.phone,
        c.active,
        c.create_date,
       now()         AS start_date,
       now()         AS end_date
FROM customer c
JOIN address a  ON (c.address_id = a.address_id)
JOIN city ci    ON (a.city_id = ci.city_id)
JOIN country co ON (ci.country_id = co.country_id);
```

Inserting data from film, language into dimMovie
```sql
INSERT INTO dimMovie (movie_key, film_id, title, description, release_year, language, rental_duration, length, rating, special_features)
SELECT 
    f.film_id as movie_key,
    f.film_id,
    f.title, 
    f.description,
    f.release_year,
    l.name as language,
    f.rental_duration,
    f.length,
    f.rating,
    f.special_features
FROM film f
JOIN language l              ON (f.language_id=l.language_id)
```

Inserting data from staff, address, city and country tables into dimStores table
```sql
INSERT INTO dimStore (store_key, store_id, address, address2, district, city, country, postal_code, manager_first_name, manager_last_name, start_date, end_date)
SELECT
    s.store_id as store_key,
    s.store_id,
    a.address,
    a.address2,
    a.district,
    c.city,
    co.country,
    a.postal_code,
    st.first_name as manager_first_name,
    st.last_name  as manager_last_name,
    now() as start_date,
    now() as end_date
FROM store s
JOIN staff st     ON    (s.manager_staff_id = st.staff_id)
JOIN address a    ON    (s.address_id = a.address_id)
JOIN city c       ON    (a.city_id = c.city_id)
JOIN country co   ON    (c.country_id = co.country_id);
```

Finally, inserting data from payment, rental and inventory tables into factSales table

```sql
INSERT INTO factSales (date_key, customer_key, movie_key, store_key, sales_amount)
SELECT 
        DISTINCT(TO_CHAR(payment_date :: DATE, 'yyyMMDD')::integer) as date_key,
        p.customer_id  as customer_key,
        i.film_id as movie_key,
        i.store_id as store_key,
        p.amount as sales_amount
FROM payment p 
JOIN rental r ON (p.rental_id = r.rental_id)
JOIN inventory i ON (r.inventory_id = i.inventory_id);
```

### Analysis and time comparison
I ran two queries on star schema to see if there's a difference in performace

star schema
```sql
SELECT dimMovie.title, sum(sales_amount) as revenue
FROM factSales 
JOIN dimMovie    on (dimMovie.movie_key      = factSales.movie_key)
GROUP BY dimMovie.title
ORDER BY revenue desc;
```



relational model
```sql
SELECT f.title, sum(p.amount) as total_amount
FROM payment as p
JOIN rental as r on p.rental_id = r.rental_id
JOIN inventory as i on r.inventory_id = i.inventory_id
JOIN film as f on i.film_id = f.film_id
GROUP BY f.title
ORDER BY total_amount desc;
```

star schema
```sql
SELECT dimMovie.title, dimDate.month, dimCustomer.city, sum(sales_amount) as revenue
FROM factSales 
JOIN dimMovie    on (dimMovie.movie_key      = factSales.movie_key)
JOIN dimDate     on (dimDate.date_key         = factSales.date_key)
JOIN dimCustomer on (dimCustomer.customer_key = factSales.customer_key)
GROUP BY (dimMovie.title, dimDate.month, dimCustomer.city)
ORDER BY dimMovie.title, dimDate.month, dimCustomer.city, revenue desc;
relational model
```

relational model
```sql
SELECT f.title, EXTRACT(month FROM p.payment_date) as month, ci.city, sum(p.amount) as revenue
FROM payment p
JOIN rental r    ON ( p.rental_id = r.rental_id )
JOIN inventory i ON ( r.inventory_id = i.inventory_id )
JOIN film f ON ( i.film_id = f.film_id)
JOIN customer c  ON ( p.customer_id = c.customer_id )
JOIN address a ON ( c.address_id = a.address_id )
JOIN city ci ON ( a.city_id = ci.city_id )
GROUP BY (f.title, month, ci.city)
ORDER BY f.title, month, ci.city, revenue desc;
```

I might not seem a lot, but when managing more data it makes a huge difference

### Loading onto Snowflake

-----------------------------------------------------------------------------------------
## Summary
This project utilizes Git and Azure cloud services to construct a data pipeline for extracting, transforming, and loading data from the 2021 Olympics in Tokyo. The pipeline involves stages such as data extraction using Git Bash, data integration with Azure Data Factory, data transformation using Azure Databricks with PySpark, and data analytics using Azure Synapse Analytics.

Throughout the project, actions were taken to address issues such as setting permissions, resolving errors, and ensuring secure access to sensitive data using Key Vault. SQL queries were used in Synapse Analytics to analyze the data, including counting athletes from each country, calculating total medals won by each country, and determining the average number of entries by gender for specific disciplines.

The project was a great opportunity for me to learn about Git, various Azure Services, as well as infrustructure and ETL process itself.

## Authors

- [@Szymon Poparda](https://www.linkedin.com/in/szymon-poparda-02b96a248/)






