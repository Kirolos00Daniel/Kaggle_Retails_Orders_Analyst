# Kaggle_Retails_Orders_Analyst

# Data Analysis and SQL Insights with Python and SQL Server

This project demonstrates the process of downloading, manipulating, and analyzing sales data using Python and SQL Server. The dataset is obtained from Kaggle, preprocessed using Python, and then analyzed using SQL queries to gain business insights.

## Prerequisites

- Python 3.x
- pandas
- SQLAlchemy
- pyodbc
- Microsoft SQL Server
- Kaggle API

## Project Steps

### 1. Kaggle API Setup and Data Download

```python
import os
from kaggle.api.kaggle_api_extended import KaggleApi

# Set up Kaggle API
api = KaggleApi()
api.authenticate()

# Download the dataset
api.dataset_download_file('ankitbansal06/retail-orders', file_name='orders.csv', path='.')
```
**Explanation:** This code sets up the Kaggle API, authenticates the user, and downloads the specified dataset ('orders.csv') from the 'retail-orders' dataset on Kaggle.

### 2. Extracting CSV File from Zip Archive

```python
import zipfile

# Extract file from zip file
zib_ref = zipfile.ZipFile('orders.csv.zip')
zib_ref.extractall()
zib_ref.close()
```
**Explanation:** Extracts the 'orders.csv' file from the downloaded 'orders.csv.zip' file using the `zipfile` module.

### 3. Reading and Cleaning the Data

```python
import pandas as pd

# Read data from the file and handle null values
df = pd.read_csv('orders.csv', na_values=['Not Available', 'unknown'])
df.head(10)
df['Ship Mode'].unique()
```
**Explanation:** Reads the CSV file into a pandas DataFrame, replaces 'Not Available' and 'unknown' values with NaN, and displays the unique values in the 'Ship Mode' column.

### 4. Renaming Columns for Consistency

```python
# Rename columns names to lowercase and replace space with underscore
df.columns = df.columns.str.lower()
df.columns = df.columns.str.replace(' ', '_')
df.head(5)
```
**Explanation:** Converts column names to lowercase and replaces spaces with underscores for consistency and easier manipulation.

### 5. Creating New Analytical Columns

```python
# Derive new columns: discount, sale price, and profit
df['discount'] = df['list_price'] * df['discount_percent'] * 0.01
df['sales_price'] = df['list_price'] - df['discount']
df['profit'] = df['sales_price'] - df['cost_price']
df.head(5)
```
**Explanation:** Calculates new columns:
- `discount`: based on the discount percentage and list price.
- `sales_price`: final price after applying the discount.
- `profit`: difference between the sales price and cost price.

### 6. Converting Date Format

```python
# Convert order date from object data type to datetime
df['order_date'] = pd.to_datetime(df['order_date'], format='%Y-%m-%d')
df.head(5)
```
**Explanation:** Converts the 'order_date' column from string format to datetime format for easier date manipulation and analysis.

### 7. Dropping Unnecessary Columns

```python
# Drop cost price, list price, and discount percent columns
df.drop(columns=['discount_percent', 'cost_price', 'list_price'], inplace=True)
df.head(5)
```
**Explanation:** Removes columns that are no longer needed after creating new derived columns.

### 8. Loading Data into SQL Server

**Connecting to SQL Server:**

```python
import sqlalchemy as sal

# Load the data into SQL Server using replace option
engine = sal.create_engine('mssql+pyodbc://DESKTOP-3A03TSH/master?driver=SQL+Server+Native+Client+11.0')
conn = engine.connect()
```
**Explanation:** Establishes a connection to the local SQL Server instance using SQLAlchemy.

**Loading Data into SQL Server Table:**

```python
# Load the data into SQL Server using append option
df.to_sql('df_orders', con=conn, index=False, if_exists='append')
```
**Explanation:** Loads the processed data into the 'df_orders' table in the SQL Server database.

### 9. SQL Analyses

#### Analysis 1: Top 10 Highest Revenue Generating Products

```sql
select top 10 product_id, sum(sales_price) as sales
from df_orders
group by product_id
order by sales desc
```
**Explanation:** Finds the top 10 products generating the highest revenue by summing up the sales prices for each product and ordering them in descending order of sales.

#### Analysis 2: Top 5 Highest Selling Products in Each Region

```sql
with tophighstsales as
(
    select region, product_id, sum(sales_price) as sales
    from df_orders
    group by region, product_id
)
select * 
from 
(
    select *, row_number() over(partition by region order by sales desc) as rn
    from tophighstsales
) A
where rn <= 5
```
**Explanation:** This query identifies the top 5 highest-selling products in each region by using a window function (`row_number`) to rank products within each region based on sales.

#### Analysis 3: Month-over-Month Sales Growth Comparison (2022 vs. 2023)

```sql
with cte as (
    select year(order_date) as order_year, month(order_date) as order_month,
    sum(sales_price) as sales
    from df_orders
    group by year(order_date), month(order_date)
)
select order_month,
    sum(case when order_year = 2022 then sales else 0 end) as sales_2022,
    sum(case when order_year = 2023 then sales else 0 end) as sales_2023
from cte
group by order_month
order by order_month
```
**Explanation:** Compares the sales growth between 2022 and 2023 for each month. It uses a common table expression (CTE) to calculate the monthly sales and then aggregates the data by year and month.

#### Analysis 4: Highest Growth in Profit by Sub-Category in 2023 Compared to 2022

```sql
with cte as (
    select sub_category, year(order_date) as order_year,
    sum(sales_price) as sales
    from df_orders
    group by sub_category, year(order_date)
),
cte2 as (
    select sub_category,
    sum(case when order_year = 2022 then sales else 0 end) as sales_2022,
    sum(case when order_year = 2023 then sales else 0 end) as sales_2023
    from cte
    group by sub_category
)
select top 1 *, (sales_2023 - sales_2022)
from cte2
order by (sales_2023 - sales_2022) desc
```
**Explanation:** Finds the sub-category with the highest growth in sales from 2022 to 2023 by calculating the difference in sales between the two years and ordering the results in descending order.

#### Analysis 5: Highest Sales Month for Each Category

```sql
with tablee as (
    select category, format(order_date,'yyyyMM') as order_year_month,
    sum(sales_price) as sales
    from df_orders
    group by category, format(order_date,'yyyyMM')
)
select * 
from (
    select *, row_number() over(partition by category order by sales desc) as rn
    from tablee
)
```
**Explanation:** Determines which month had the highest sales for each category by partitioning the data by category and using the `row_number` function to rank the months based on sales.

## Future Enhancements

- **Advanced Analytics**: Implement more complex analytical queries for deeper insights.
- **Data Visualization**: Integrate with BI tools like Power BI or Tableau for advanced data visualization.
- **Automated Reporting**: Set up automated reports using Python or SQL Server Reporting Services (SSRS).

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

