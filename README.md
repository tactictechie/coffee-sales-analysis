# coffee-sales-analysis

Objective : The objective of this project is to analyze the sales data of a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new outlets based on the consumer demand and sales performance .

## Key Questions 

-- Q.1 Coffee Consumers Count
-- How many people in each city are estimated to consume coffee , given that 25% of population does

SELECT city_id ,
city_name , 
CONCAT(CAST((population*0.25)/1000000 AS DECIMAL(10,2)),' ','Million')AS coffee_drinkers_in_Millions
FROM city
ORDER BY coffee_drinkers_in_Millions

-- Q2 Total revenue from coffee sales
-- What is the total revenue generated from coffee sales accross all citites in the last quarter of 2023

SELECT ct.city_name ,
SUM(total) AS total_sales FROM Sales s
JOIN customers c
ON c.customer_id=s.customer_id
JOIN city ct 
ON ct.city_id = c.city_id
WHERE DATEPART(QUARTER,sale_date) =4
GROUP BY ct.city_name


-- Q3 Sales count for each product
-- How many units of each product has been sold

SELECT p.product_id,p.product_name,COUNT(s.sale_id) AS total_units FROM Sales s
LEFT JOIN Products p
ON p.product_id = s.product_id
GROUP BY p.product_id,p.product_name
ORDER BY total_units DESC


-- Q.4 Average sales amount per city
-- What is the average sales amount per customer in each city

SELECT ct.city_name , SUM(s.total) AS total_revenue,
COUNT(DISTINCT c.customer_id) AS total_customers
, CAST( SUM(s.total) /COUNT(DISTINCT c.customer_id)
 AS DECIMAL(10,2)) AS avg_sales_per_cust
FROM Sales s
JOIN customers c 
ON c.customer_id=s.customer_id
JOIN city ct 
ON ct.city_id = c.city_id
GROUP BY ct.city_name
ORDER BY avg_sales_per_cust

-- Q.5 City population and coffee consumers(25%)
-- Provide a list of cities along with their populations and estimated coffee consumers

SELECT ct.city_name ,
CAST(ct.population/1000000 AS DECIMAL(10,2)) AS population_in_mln,
CONCAT(CAST(ct.population*0.25/1000000 AS DECIMAL(10,2)),' ','Million') AS estimated_customers_in_mln,
COUNT(DISTINCT s.customer_id) AS current_cst
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
GROUP BY ct.city_name,
CAST(ct.population/1000000 AS DECIMAL(10,2)),
CAST(ct.population*0.25/1000000 AS DECIMAL(10,2));


-- Q.6 Top selling products by city
-- What are the top 3 selling products in each city based on sales volume
WITH cte AS (SELECT ct.city_name,p.product_name,
COUNT(s.sale_id) AS total_units,
DENSE_RANK() OVER (PARTITION BY ct.city_name ORDER BY  COUNT(s.sale_id) DESC) AS rnk
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
JOIN products p on
s.product_id = p.product_id
GROUP BY ct.city_name,p.product_name,
s.product_id )

SELECT city_name, product_name,
total_units 
FROM cte
WHERE rnk <=3
ORDER BY city_name;

-- Comma separated products 
WITH cte AS (SELECT ct.city_name,p.product_name,
COUNT(s.sale_id) AS total_units,
DENSE_RANK() OVER (PARTITION BY ct.city_name ORDER BY  COUNT(s.sale_id) DESC) AS rnk
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
JOIN products p on
s.product_id = p.product_id
GROUP BY ct.city_name,p.product_name,
s.product_id )

SELECT city_name, STRING_AGG (product_name, ', ') WITHIN GROUP( ORDER BY total_units DESC) as top_products 
FROM cte
WHERE rnk <=3
GROUP BY city_name


-- Q.7 Customer segmentation by city
-- How many unique customers are there in each city who have purchased coffee products (consumable)?

SELECT ct.city_name , COUNT(DISTINCT c.customer_id) AS total_customers 
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
JOIN products p on
s.product_id = p.product_id
WHERE s.product_id IN(1,2,3,4,5,6,7,8,9,10,11,12,13,14)
GROUP BY ct.city_name;

SELECT * FROM Products

-- Q.8 Average Sale vs rent
-- Find each city and their average sale per customer and avg rent per customer
SELECT ct.city_name , COUNT(DISTINCT s.customer_id) as customers,
CAST(SUM(total) AS DECIMAL(10,2)) AS total_revenue,
CAST(SUM(total) / COUNT(DISTINCT s.customer_id) AS DECIMAL(10,2)) AS avg_rev_per_cst,
CAST(ct.estimated_rent / COUNT(DISTINCT s.customer_id)AS DECIMAL(10,2)) AS avg_rent_per_cst
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
JOIN products p on
s.product_id = p.product_id 
GROUP BY ct.city_name, ct.estimated_rent;

-- Q.9 Monthly Sales Growth 
-- Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly) by each city
WITH cte AS(
	SELECT ct.city_name,
	MONTH(sale_date) AS month,
	YEAR(sale_date) AS year,
	SUM(total) AS current_revenue,
	LAG(SUM(total)) OVER (PARTITION BY ct.city_name ORDER BY YEAR(sale_date) ,  MONTH(sale_date) ) AS prev_month_sale 
	FROM sales s JOIN
	customers c 
	ON s.customer_id=c.customer_id
	JOIN city ct 
	ON ct.city_id=c.city_id
	GROUP BY ct.city_name, MONTH(sale_date), YEAR(sale_date)
	)

	SELECT city_name, month,year,
	current_revenue,
	CAST((current_revenue-prev_month_sale)/ prev_month_sale AS DECIMAL(10,2))  AS perc_change
	FROM cte
	WHERE prev_month_sale IS NOT NULL

-- Q.10 Market potential analysis 
-- Identify cities based on highest sales , return city name , total sale , total rent , total customer , estimated coffee consumers

SELECT  ct.city_name , SUM(s.total) AS total_sales,
ct.estimated_rent,
COUNT(DISTINCT s.customer_id) AS total_customers ,
CONCAT(CAST(ct.population*0.25/1000000 AS DECIMAL(10,2)),' ','Million') AS estimated_consumers,
CAST(SUM(total) / COUNT(DISTINCT s.customer_id) AS DECIMAL(10,2)) AS avg_rev_per_cst,
CAST(ct.estimated_rent / COUNT(DISTINCT s.customer_id)AS DECIMAL(10,2)) AS avg_rent_per_cst
FROM sales s JOIN
customers c 
ON s.customer_id=c.customer_id
JOIN city ct 
ON ct.city_id=c.city_id
JOIN products p on
s.product_id = p.product_id 
GROUP BY ct.city_name , ct.estimated_rent , ct.population
ORDER BY total_sales DESC

/* TOP 3 City Recommendations
City 1 Pune : 
				i)	 Highest revenue
				ii)	 Average rent per customer is less
				iii) Average sale per customer is also high 

City 2 Jaipur : 
				i)	 High revenue
				ii)	 Average rent per customer is very less
				iii) Average sale per customer is also high
				iv) High amount of current customers


City 2 Delhi :
				i)	 Highest consumer count
				ii)	 High amount of current customers
				iii) Average sale per customer is also high 
