# Free Trial Performance Analysis in SQL

<img src="https://i.imgur.com/t2nMfMU.png" height="50%" alt="Data Analysis Image by Mohamed Hassan from Pixabay"/>

<h2>🗺️Project Description</h2>
This is a data analysis portfolio project that aims to analyze marketing data to determine the value of a product's global free trial promotion. The project involves summarizing and aggregating data to calculate key metrics using SQL and a PostgreSQL database.

<h2>💻Languages/Libraries and Utilities Used</h2>

- <b>SQL</b> 
- <b>PostgreSQL</b>

<h2>📝Dataset Description:</h2>
Synthetic data as four tables created by DataCamp. The data represents a product sold through a 1-month trial. The Free Trials table records instances of customers beginning a free trial. One month after the free trial period starts, the customer may choose to pay, and if so we will have a Purchase record. These are the 4 tables:

- <b>Free Trials</b> - List of instances of free trial starts.
- <b>Purchases</b> - List of instances of customers paying, following their free trial.
- <b>Dates</b> - List of dates, provided for convenience.
- <b>Prices</b> - List of prices of the product by region over time. 

<h2>🧹Getting Familiar with the Data:</h2>

- Group `trials` by the month of `free_trial_start_date` (and order by the same). Count the rows as `num_free_trials`.
```bash
select
        date_trunc('month', free_trial_start_date) as month
    ,	count(*) as num_free_trials
from trials
    group by 1
    order by 1
```
<img src="https://i.imgur.com/Fm09A6h.png" height="40%" alt="Table - Number of Free Trials by Month"/>

- Group `purchases` by the month of `purchase_date` (and order by the same). Count the rows as `num_purchases`, and sum `purchase_value` as `usd_value`. Call the output `purchases_by_month`.
```bash
select
        date_trunc('month', purchase_date) as month
    ,	count(*) as num_purchases
    ,	sum(purchase_value) as usd_value
from purchases
    group by 1
    order by 1
```
<img src="https://i.imgur.com/APTPPW8.png" height="40%" alt="Table - Group Purchases by Month with Aggregations"/>

- Create a line graph of `num_purchases` by `month`. 
<img src="https://i.imgur.com/WwDQ6eX.png" height="80%" alt="Line Graph - Number of Purchases per Month"/>

<h2>🚄Data Aggregation 1 - Velocity Metrics by Month</h2>

- Aggregate data by month, create summaries as Common Table Expressions (CTEs), and outer join purchases per month against the free trials per month to get the results into a combined results table.
```bash
with free_trials_per_month as (
    select
            date_trunc('month', free_trial_start_date) as month
        ,	count(*) as num_free_trials
    from trials
        group by 1
        order by 1
)

,	purchases_per_month as (
    select
            date_trunc('month', purchase_date) as month
        ,	count(*) as num_purchases
        ,	sum(purchase_value) as usd_value
    from purchases
        group by 1
        order by 1
)

select
		coalesce(free_trials_per_month.month, purchases_per_month.month) as month
    ,	free_trials_per_month.num_free_trials
    ,	purchases_per_month.num_purchases
    ,	purchases_per_month.usd_value
from free_trials_per_month
	full join purchases_per_month
    	on purchases_per_month.month = free_trials_per_month.month
```
<img src="https://i.imgur.com/h3fHZlh.png" height="80%" alt="Table - Joining Purchases Per Month and Free Trails"/>

<h2>🟥🟧🟨Data Aggregation 2 - Cohort Metrics by Month</h2>

- Select all columns in `trials`, and left join the columns in `purchases` on their shared `trial_id` column.
```bash
select
        trials.trial_id
    ,	trials.free_trial_start_date
	,	trials.region
    ,	purchases.purchase_date
	,	purchases.purchase_value
from trials
	left join purchases
    	on purchases.trial_id = trials.trial_id
```
<img src="https://i.imgur.com/8f5dC2a.png" height="40%" alt="Table - Join Purchases On Shared Trial ID Column"/>

- Aggregate all the data by the month of the Free Trial start, and calculate the same metrics as before; `num_free_trials`, `num_purchases` and `usd_value`.
```bash
with free_trials_and_purchases as (
	select
            trials.trial_id
        ,	trials.free_trial_start_date
        ,	trials.region
        ,	purchases.purchase_date
        ,	purchases.purchase_value
    from trials
        left join purchases
            on purchases.trial_id = trials.trial_id
)

select
		date_trunc('month', free_trial_start_date) as month
    ,	count(*) as num_free_trials
    ,	count(purchase_date) as num_purchases -- Count how many dates exist to see how many purchases occurred.
    ,	sum(purchase_value) as usd_value
from free_trials_and_purchases
        group by 1
        order by 1
```
<img src="https://i.imgur.com/mgb5KCa.png" height="40%" alt="Table - Joining Purchases Per Month and Free Trails"/>

<h2>🆓Free Trial Value Calculation</h2>
Calculate an average value per Free Trial per month using the Cohort table previously created.

- Take the previous aggregation and create an additional metric called `cohort_value_per_free_trial` by dividing `purchase_value` by `num_free_trials`.
```bash
with free_trials_and_purchases as (
	select
            trials.trial_id
        ,	trials.free_trial_start_date
        ,	trials.region
        ,	purchases.purchase_date
        ,	purchases.purchase_value
    from trials
        left join purchases
            on purchases.trial_id = trials.trial_id
)

,	summary_by_month as (
    select
            date_trunc('month', free_trial_start_date) as month
        ,	count(*) as num_free_trials
        ,	count(purchase_date) as num_purchases
        ,	sum(purchase_value) as usd_value
    from free_trials_and_purchases
            group by 1
)

select
		month
	,	num_free_trials
    ,	num_purchases
    ,	usd_value
    ,	(usd_value::float) / (nullif(num_free_trials, 0)::float) as cohort_value_per_free_trial -- To avoid dividing by zero, replacing any zero values in the denominator with NULL. Also convert to FLOAT before dividing to ensure the result is accurate.
from summary_by_month
	order by 1 
```
<img src="https://i.imgur.com/t5dm9h0.png" height="40%" alt="Table - Cohort Value Per Free Trial"/>

<h2>📈Dimensional Breakdown</h2>
Break down average value per Free Trial by Region and analyze differences in values.

- Introduce and group by the additional dimension: `region`. Call the resultant table `cohort_value_by_month_and_region.

```bash
with free_trials_and_purchases as (
	select
            trials.trial_id
        ,	trials.free_trial_start_date
        ,	trials.region
        ,	purchases.purchase_date
        ,	purchases.purchase_value
    from trials
        left join purchases
            on purchases.trial_id = trials.trial_id
)

,	summary_by_month as (
    select
            date_trunc('month', free_trial_start_date) as month
    	,	region
        ,	count(*) as num_free_trials
        ,	count(purchase_date) as num_purchases
        ,	sum(purchase_value) as usd_value
    from free_trials_and_purchases
            group by 1, 2
)

select
		month
    ,	region
	,	num_free_trials
    ,	num_purchases
    ,	usd_value
    ,	(usd_value::float) / (nullif(num_free_trials, 0)::float) as cohort_value_per_free_trial
from summary_by_month
	order by 1, 2
```
<img src="https://i.imgur.com/Cj9vnak.png" height="40%" alt="Table - Cohort Value Per Free Trial"/> 

- Create a graph of `cohort_value_per_free_trial`.
<img src="https://i.imgur.com/NEwVyDv.png" height="80%" alt="Line Graph Cohort Value Per Free Trial by Month and Region"/>

<h2>🕹️Visualization of Findings:</h2>

- A tourist is coming from the U.K. and looking to book an Amsterdam Airbnb in their GBP currency
<img src="https://i.imgur.com/6am49tZ.png[/img]" height="80%" alt="Amsterdam Dataframe for Recommendation System"/>

- You can see this same dataset as an interactive, geographic visualization [by visiting my Streamlit](https://heyshatara-numpy-airbnb-streamlit-app-gzn89d.streamlit.app/)

 ```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
--!>
