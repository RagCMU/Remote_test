# Senior Product Analyst Home Assignment


### Dataset

The original Google sheet file shared [Senior Data Analyst Assignment](https://docs.google.com/spreadsheets/d/1QJskiqfsriaoQXIEp7rrC0ZtMM-Y5z7PKxBAiw8HQdo/edit#gid=1921694854) had 5 CSV files : DimCustomer, DimProduct,DimSalesTerritory, FactResellersSales and DimInvoices. To make it easier to work with I parsed these files using Python in order to create Pandas dataframes named **dimcusomter, _dimproduct_, dimsales, factresllers, diminvoices** . The entire process can be viewed in my data pre-processing jupyter notebook here  [Data Preprocesing and Cleaning Notebook](https://nbviewer.org/github/RagCMU/Remote_test/blob/main/Data%20pre-processing.ipynb)

Below is a short summary of the pre-processing task I have perfomed on the raw datasets :

* Loaded the individual CSV files into a pandas dataframes
* Inspected the CSV files for column names, datatypes and number of rows and columns
* Checked for NULL values and replaced NULL values with appropriate values as per the column data types.
* Checked for duplicate values for primary keys
* FactResellersSales dataset did not have a unique identifier to identify each transaction hence created a composite primary key by combining *SalesOrderLineNumber* and *SalesOrderNumber*



## 1. Database Connection

Once satisfied with Data quality , I proceeded to create the database connection. I used the Python SQLite data library to open database connection and named the database as **Remote_test.db**. 
Used python function *to_sql * to insert the CSV files in to a SQLite database named Remote_test database. This standalone database file was then imported using a SQL IDE DBeaver using a SQLite connection
The steps and details can be dound in the  [Data Preprocesing and Cleaning Notebook](https://nbviewer.org/github/RagCMU/Remote_test/blob/main/Data%20pre-processing.ipynb)


You can find the database file here in my Github repo : [Remote Assignment](https://github.com/RagCMU/Remote_test/blob/main/remote_test.db)

![SQLLite Database connection](https://github.com/RagCMU/Remote_test/blob/main/Screenshot%202024-05-02%20at%2010.58.02%20pm.png)

[SQLLite Database connection](https://github.com/RagCMU/Remote_test/blob/main/Screenshot%202024-05-02%20at%2011.00.03%20pm.png)

In order to connect to it you would have to download the file and create a new SQL Lite connection using the IDE of your choice, as shown above in the screenshots above.

*Note : My preferred method to create the database would have been through MySQL or PostgreSQL but I ran into database connection issues which took me longer than expected time to debug.*


## 2. Exploratory Analysis

I had already performed some basic exploratory analysis in Pandas to understand the given dataset. Using SQL I further investigated the data to get a deeper understanding of the differrent characteristics and relationships within a dataset.


**PRODUCTS**

`SELECT COUNT (DISTINCT ProductKey) AS Unique_products,
COUNT(DISTINCT ProductName) AS Uniqueproduct_names
FROM dim_product`


Unique_products 569 
Uniqueproduct_names 488


We can see that although unique product_ids are less than the product names indicating that some of the products may have more than 1 product id. 
On further investigation we come to know that each product name has more than 1 version and all version of every product are uniquely identified by the product key. These different product version can be differentiated based on their different start and end times of use.

`SELECT productname,startdate,enddate FROM dim_product 
WHERE productname = 'Sport-100 Helmet, Red'`

`
Sport-100 Helmet, Red	2011-07-01 00:00:00	2007-12-28 00:00:00
Sport-100 Helmet, Red	2012-07-01 00:00:00	2008-12-27 00:00:00
Sport-100 Helmet, Red	2013-07-01 00:00:00	  
`



**TERRITORY**

Let’s see which territories are generating the maximum sales and revenue : 

`/* Total Sales and revenue  by Territory*/
SELECT *
FROM (
SELECT SalesTerritoryCountry as Territory
,COUNT(SalesTerritoryCountry) AS total_sales
,ROUND(SUM(SalesAmount),2) AS total_revenue
 FROM dim_sales ds
 INNER JOIN fact_resellers fr
 ON ds.SalesTerritoryKey = fr.SalesTerritoryKey
  GROUP BY SalesTerritoryCountry
  ORDER BY total_revenue  DESC`

      `
    Territory      Total sales   Total revenue
    United States	38809	     53607801.21
    Canada	        11444	     14377925.6
    France	        3530	     4607537.93
    United Kingdom	3520	     4279008.83
    Germany	        1839	     1983988.04
    Australia	    1713	     1594335.38
    `

**INVOICES**

Let’s do summary statistics on the invoices : 

`
SELECT COUNT(DISTINCT invoice_id) AS total_invocies
,COUNT( DISTINCT CASE WHEN invoice_status= 'invoiced' THEN invoice_id ELSE NULL END) AS invoiced,
COUNT( DISTINCT CASE WHEN invoice_status= 'pending' THEN invoice_id ELSE NULL END) AS pending,
SUM( CASE WHEN invoice_status= 'invoiced' THEN discounted_amount_usd  ELSE 0  END) AS   inv_disc_amount,
SUM( CASE WHEN invoice_status= 'pending' THEN discounted_amount_usd  ELSE 0  END) AS pending_disc_amount
FROM dim_invoices
`

  `
  total_invoices  invoiced_   pending   cancelled    inv_disc_amount    pending_disc_amount
  500	            372        	90	      38	         396574.45	         26093.1
  `


## 3. SQL Questions 

For each of the questions below, I have added comments at every step which should explain how I arrived at the final solution.

#### Question 1: What is the highest transaction of each month in 2012 for the product Sport-100 Helmet, Red?

```
-- Get details from the dim_product table for the product named 'Sport-100 Helmet, Red'

With product AS
(
SELECT productkey,productalternatekey,productname,startdate,enddate FROM dim_product 
WHERE productname = 'Sport-100 Helmet, Red'
)

-- Seems like we have 3 uniques product_keys with the same name 'Sport-100 Helmet, Red' but these are same product 
-- with different versions dinstinguished by their productkey and startdate and endate.


-- Lets first get the list of all monthly transaction for the 'Sport-100 Helmet, Red' products sold for the year 2012 , 
-- this will set us up to retrieve the highest transaction each
-- NOTE : I have included product key to know which version of the product was sold
-- SalesOrderNumber will provide us with unique sale id for each of the product sold.

,

transactions AS 
(
SELECT distinct SalesOrderNumber
,fr.productkey
,productname,strftime('%m-%Y', Orderdate) as month_year,orderdate
,ROUND(SalesAmount,2) as SalesAmount
FROM fact_resellers fr 
INNER JOIN product p ON fr.productkey = p.productkey
WHERE Orderdate >= '2012-01-01 00:00:00' AND Orderdate < '2013-01-01 00:00:00'
)

--- Now ,in order to find the highest transaction let's use rank() or dense_rank() function 

SELECT month_year
,SalesAmount
,Orderdate FROM
(select *, dense_rank() over (partition by month_year order by SalesAmount desc) as rank from transactions)
where rank = 1`
```


#### Question 2: Find all the products and their total sales amount by month. Only consider products that had at least one sale in 2012.

```
-- Get details from the dim_product table for all product and join with fact_resellers for monthly sales amount


SELECT 
p.productname
,strftime('%m-%Y', Orderdate) as month_year
,ROUND(SUM (SalesAmount),2) as total_sales_amount
from fact_resellers fr 
INNER JOIN dim_product  p ON fr.productkey = p.productkey
WHERE Orderdate >= '2012-01-01 00:00:00' AND Orderdate < '2013-01-01 00:00:00'
group by 1,2
order by 2,1
```

#### Question 3: What is the combination of product & country with the lowest amount of sales per month in 2012?

The way I have interpreted this question is the the product-country combination with lowest sales per month across all months of 2012.


```
-- In order to get the sales per month for 2012 for all productsand terittories lets join
-- sales data with dim_product and dim_sales

WITH sales as
(
SELECT 
strftime('%m-%Y', Orderdate) as month_year
,p.productname
,ds.SalesTerritoryCountry 
,CONCAT (p.productname,'-',ds.SalesTerritoryCountry) as product_sales_territory -- single colum with product territory combined
,ROUND(SUM (SalesAmount),2) as total_sales_amount
from fact_resellers fr 
INNER JOIN dim_product  p ON fr.productkey = p.productkey
INNER JOIN dim_sales ds ON ds.SalesTerritoryKey = fr.SalesTerritoryKey 
WHERE Orderdate >= '2012-01-01 00:00:00' AND Orderdate < '2013-01-01 00:00:00'
group by 1,2,3,4
order by 2,1
)

,

/* Now we have the sales per month data for every combination of product and teritory 
 * using this dataset we should be able to rank the product and teritory  with the lowest sales amount for every month
 * in 2012
 */


sales_month AS 
(
SELECT month_year, productname,SalesTerritoryCountry,total_sales_amount from 

(SELECT *, DENSE_RANK() OVER (PARTITION BY month_year ORDER BY total_sales_amount ) as rank from sales)
where rank = 1
)



-- Finally select the product and sales territory combo with the lowest sales per month in 2012 

SELECT month_year, productname,SalesTerritoryCountry,total_sales_amount FROM sales_month
ORDER BY 4
Limit 1
```


## 4. Problem Solving Exercise

For this exercise I  used  Miro to create the KPI Driver tree. I have written a short explanation paragraph on how I arrived at the final KPI’s for boosting Remote talent platform hiring rate , you can find 
the high level driver tree here along with the explanation :  [Remote Talent KPI Driver Tree](https://miro.com/app/board/uXjVKNOAHVY=/)


