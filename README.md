# 👟 Adidas Sales Project 

Project for the analysis of Adidas US sales data from 2021 using PostgreSQL for in-depth analysis, followed by Tableau for comprehensive data visualization.

## Background
Sales are a fundamental aspect of any business. 

It's a direct link to a company's revenue as it offers insights into what's working, what's not, and where a company can improve. But it also gives you a clue of where your most loyal customers are.

In this project, I'll analyze two tables repurposed from the [Adidas Sales Dataset](https://www.kaggle.com/datasets/heemalichaudhari/adidas-sales-dataset) in Kraggle to answer the following questions:

1. What is the revenue generated by each product, ranked from highest to lowest?
2. What are the top three products on highest demand in terms of quantity?
3. Which retailers have the most sales per state?
4. Are there any interesting patterns as to when customers buy more products in terms of seasonality?

By correlating insights from this sales data with marketing efforts, Adidas can allocate marketing budgets more effectively and generate more revenue.

**NOTE: The methodology section provides the project's background. Within the results, you'll find code snippets and data outputs from PostgreSQL along with answers to the project's questions. The data visualization segment showcases the Tableau dashboard. Finally, the recommendations section gives you the next steps for Adidas**

## 📊 Methodoly

I started off with two Excel files. 

The first Excel file, Adidas US Sales (as its name suggests), had information on transaction id, retailer, retailer id, invoice date, product, prices per unit, units sold, and sales method. 

The second Excel file, Adidas US Locations, had information on transaction id, region, state, and city. 

It's clear from the beginning I have to import both of these files to PostgreSQL as tables and then join them as one. This would help further analysis to answer questions that have to do with the states that generated the most sales as well as trends.

Here's the step-by-step of the methodology I followed:

### Creating empty tables

In order to import the information on the Excel files into PostgreSQL, I had to create an empty table for each Excel sheet. These empty tables must contain the same column name of the headers while keeping in mind the data type, and column constraints.

I created the first table "adidas_sales" with the following code:

```
CREATE TABLE adidas_sales (
	transaction_id SERIAL PRIMARY KEY,
	retailer VARCHAR(50) NOT NULL,
	retailer_id INTEGER NOT NULL,
	invoice_date TIMESTAMP NOT NULL,
	product VARCHAR(50) NOT NULL,
	price_per_unit INTEGER NOT NULL,
	units_sold INTEGER NOT NULL,
	total_sales INTEGER NOT NULL,
	sales_method VARCHAR(50) NOT NULL
);
```

Then I created the second empty table "locations" with this query:

```
CREATE TABLE locations (
	transaction_id INTEGER REFERENCES adidas_sales(transaction_id),
	region VARCHAR(25) NOT NULL,
	state VARCHAR(25) NOT NULL,
	city VARCHAR(25) NOT NULL
);
```

Once I created both tables I refreshed the database and selected everything (SELECT *) from each table to check if the names of the columns with no information were there. Then I saved the two Excel files as CVS files and imported them into their corresponding table in PostgreSQL.

A quick **SELECT * FROM** adidas_sales and locations showed me that all the information was there!

### Joining tables

The information I needed to solve all the questions above is in both adidas_sales and locations. Since both of these have the transaction_id column in common, I can use the command **INNER JOIN** to analyze data from just one "master" table.


```
SELECT * FROM adidas_sales
INNER JOIN locations
ON adidas_sales.transaction_id = locations.transaction_id;
```

With this "master" table you can go ahead and perform more specific commands to solve the questions for this project.

## 💡 Results

### What is the revenue generated by each product, ranked from highest to lowest?

To find the revenue generated for each product you have to multiply the price per unit by the units sold and create a new column with the name "total_sales". Also, by reviewing the product data and using the command **DISTINCT** you can tell that there are six main categories of products which are:

- "Men's Street Footwear"
- "Women's Athletic Footwear"
- "Men's Athletic Footwear"
- "Women's Apparel"
- "Women's Street Footwear"
- "Men's Apparel"

This means that before I order the revenue generated by product (or category of product) I'll have to group by product.

If I want to calculate the total sales for each product and group by the product, I'll have to use a subquery to first calculate the total sales, and then perform the grouping.

Here's the SQL query for that:

```
SELECT product, SUM(total_sales) AS total_sales
FROM (SELECT adidas_sales.product,
    (adidas_sales.price_per_unit * adidas_sales.units_sold) AS total_sales
  FROM adidas_sales
  INNER JOIN locations
  ON adidas_sales.transaction_id = locations.transaction_id
) AS subquery
GROUP BY product
ORDER BY total_sales DESC;
```

And this is the data output:

<img width="230" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/04775ecb-837e-4d48-95fe-7d331f38eeef">

### What are the top three products on highest demand in terms of quantity?

In this case, I'll have to use an aggregate function (again) to create a new column called total_units_sold, then group by product and order by the new column.

Here's the SQL query:

```
SELECT product, SUM(units_sold) AS total_units_sold FROM adidas_sales
INNER JOIN locations
ON adidas_sales.transaction_id = locations.transaction_id
GROUP BY product
ORDER BY total_units_sold DESC
LIMIT 3;
```

And the data output:

<img width="239" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/cfd1b4f1-753f-4038-840b-945c81a532b5">

### Which retailers have the most sales per state?

To answer this question, I'll need to perform the following steps:

1. Calculate the total sales for each retailer and state combination
2. Rank the retailers within each state based on their total sales
3. Select the retailer with the highest total sales in each state

Here's the SQL query for that:

```
WITH retailer_state_sales AS (
  SELECT locations.state, adidas_sales.retailer,
    SUM(adidas_sales.price_per_unit * adidas_sales.units_sold) AS total_sales
  FROM adidas_sales
  INNER JOIN locations
  ON adidas_sales.transaction_id = locations.transaction_id
  GROUP BY locations.state, adidas_sales.retailer
),
ranked_retailers AS (
  SELECT state, retailer, total_sales,
    ROW_NUMBER() OVER (PARTITION BY state ORDER BY total_sales DESC) AS rank
  FROM retailer_state_sales
)
SELECT state, retailer, total_sales
FROM ranked_retailers
WHERE rank = 1;
```
And here's a screenshot of the retailers with the most sales in the first 10 states by alphabetical order:

<img width="332" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/a52ce4f7-82d8-4e6a-94bf-b280a1961320">

### Are there any interesting patterns as to when customers buy more products in terms of seasonality?

To answer this question, it would be more helpful to have a dashboard in Tableau that allowed us to see these trends in seasonality (more on that in the following section 😉). For now, I can analyze the seasonality of sales by grouping the sales data by seasons (winter, spring, summer, and fall). Then you can calculate the total sales for each season.

Here's the SQL query:

```
SELECT 
  CASE 
    WHEN EXTRACT(MONTH FROM invoice_date) IN (12, 1, 2) THEN 'Winter'
    WHEN EXTRACT(MONTH FROM invoice_date) IN (3, 4, 5) THEN 'Spring'
    WHEN EXTRACT(MONTH FROM invoice_date) IN (6, 7, 8) THEN 'Summer'
    WHEN EXTRACT(MONTH FROM invoice_date) IN (9, 10, 11) THEN 'Fall'
  END AS season, SUM(price_per_unit * units_sold) AS total_sales
FROM adidas_sales
GROUP BY season
ORDER BY total_sales;
```

And here's the data output:

<img width="164" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/d4a1d427-4f74-4956-b288-b81eb21ffa19">

## ✨ Data Visualization in Tableau

**Check the Adidas Sales Dashboard in Tableau [here](https://public.tableau.com/app/profile/jessica.jansasoy/viz/AdidasSalesDashboard_16977627312930/Dashboard1).**

Using PostgreSQL was effective in answering specific questions. However, a data visualization tool it's a better option to highlight insights that may not be immediately apparent when looking at rows and columns of numbers (for example the question about seasonality). Plus, with visualizations, it's easier to compare datasets or elements within a dataset.

Comparing sales figures over different time periods or evaluating the performance of different products is going to become clearer.

For this project, I needed graphics that showed the following:

- The portion of revenue each product category brings to Adidas
- A trend line to identify seasonal trends and changes during the year
- The total sales per state in a more visual way
- And (as a bonus) a graphic with information about sales by retailer and their sales method

Here are the results:

### Figure 1. Total Sales by Product Category

- **Visualization type:** Pie chart
- **Dimensions:** Product Category
- **Measure:** SUM(Total Sales)
- **Description:** In this pie chart, for Figure 1, you can easily identify which product categories contribute the most to total sales. In this case, is the Men's Street Footwear.

<img width="402" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/2fd227ed-415e-4ef4-a3a3-ac675a22ddef">

### Figure 2. Total Sales and Units Sold per Month

- **Visualization type:** Line chart
- **Dimensions:** Month(Invoice Date)
- **Measure:** SUM(Total Sales) and SUM(Units Sold)
- **Description:** In this line chart, for Figure 2, you can check for seasonal trends. In this case, there's a peak in sales in July while the minimum value of sales is in March. As a bonus, I included a line chart for units sold, which is in sync with the sales. 

<img width="521" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/ebc0e0ad-fadf-4427-8138-fb90c898fd1b">

### Figure 3. Total Sales by State

- **Visualization type:** Horizontal bar chart
- **Dimensions:** State
- **Measure:** SUM(Total Sales)
- **Description:** In this horizontal bar chart, for Figure 3, you have a clear view of which states have the highest revenue in sales. In this case, California generates de most sales.

<img width="475" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/351daadb-dfa9-4962-8698-9d7ee0f57c2d">


### Figure 4. Total Sales by Retailer and Sales Method

- **Visualization type:** Stacked bar chart
- **Dimensions:** Retailer and Sales Method
- **Measure:** SUM(Total Sales) (yes, this again!)
- **Description:** In this stacked bar chart, for Figure 4, you can see the distribution of sales across different methods and identify which one is more effective for a specific retailer.

<img width="491" alt="image" src="https://github.com/jjansasoy/adidas_sales/assets/63824959/edd4f173-409f-48ee-94ec-475dce6574e0">

## 🧠 Recommendations





