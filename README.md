# Free Trial Performance Analysis in SQL

<img src="https://i.imgur.com/vr42hjE.png" height="50%" alt="Amsterdam's De Gooyer Windmill"/>

<h2>🗺️Project Description</h2>
This is a (marketing) data analysis portfolio project that aims to analyze marketing data for a free trila of a product. The project involves summarizing and aggregating data to calculate key metrics SQL and a PostgreSQL database.

<h2>💻Languages/Libraries and Utilities Used</h2>

- <b>SQL</b> 
- <b>PostgreSQL</b>

<h2>📝Dataset Description:</h2>
Synthetic data as four tables created by DataCamp. The data represents a product sold through a 1-month trial. The Free Trials table records instances of customers beginning a free trial. One month after the free trial period starts, the customer may choose to pay, and if so we will have a Purchase record. These are the 4 tables:

- <b>Free Trials</b> 
- <b>Purchases</b> 
- <b>Dates</b> 
- <b>Prices</b>

<h2>🧹Getting Familiar with the Data:</h2>

- Query Free Trials & Purchases tables.
```bash
select
        date_trunc('month', free_trial_start_date) as month
    ,	count(*) as num_free_trials
from trials
    group by 1
    order by 1
```
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
- Create a line graph of `num_purchases` by `month`. 
<img src="https://i.imgur.com/WwDQ6eXW.png" height="80%" alt="Line Graph - Number of Purchases per Month"/>

# Remove the first column and row

<h2>💱Data Aggregation 1 - Velocity Metrics by Month</h2>
The Amsterdam team at Airbnb wants to provide recommendations for listings based on proximity to their destination point(s). But to make this more insightful, the rates for each listing needed to be converted to reflect the tourist's desired currency.
We created and tested this for tourists from the United Kingdom who would be booking in their GBP currencies.

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
<h2>💱Data Aggregation 2 - Cohort Metrics by Month</h2>
The Amsterdam team at Airbnb wants to provide recommendations for listings based on proximity to their destination point(s). But to make this more insightful, the rates for each listing needed to be converted to reflect the tourist's desired currency.
We created and tested this for tourists from the United Kingdom who would be booking in their GBP currencies.

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
    ,	count(purchase_date) as num_purchases -- We count how many dates exist to see how many purchases occurred.
    ,	sum(purchase_value) as usd_value
from free_trials_and_purchases
        group by 1
        order by 1
```
<h2>💱Free Trial Value Calculation</h2>
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
    ,	(usd_value::float) / (nullif(num_free_trials, 0)::float) as cohort_value_per_free_trial -- It's important to avoid dividing by Zero, so we replace any zero values in the denominator with NULL. We also convert to FLOAT before dividing to ensure that the result is accurate.
from summary_by_month
	order by 1 -- Order By only matters on the last statement.
```
<h2>💱Dimensional Breakdown</h2>
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
- Create a graph of `cohort_value_per_free_trial`.
<img src="https://i.imgur.com/NEwVyDv.png" height="80%" alt="Line Graph Cohort Value Per Free Trial by Month and Region"/>

<h2>🕹️Visualization of Findings:</h2>

- A tourist is coming from the U.K. and looking to book an Amsterdam Airbnb in their GBP currency
<img src="https://i.imgur.com/6am49tZ.png[/img]" height="80%" alt="Amsterdam Dataframe for Recommendation System"/>

- You can see this same dataset as an interactive, geographic visualization [by visiting my Streamlit](https://heyshatara-numpy-airbnb-streamlit-app-gzn89d.streamlit.app/)

- All of the Amsterdam Airbnb listings are shown in red to gauge their proximity to a specific tourist spot shown in blue, which is set as De Hooyer Windmill in Amsterdam:


<!--
 ```diff
- text in red
+ text in green
! text in orange
# text in gray
@@ text in purple (and bold)@@
```
--!>
