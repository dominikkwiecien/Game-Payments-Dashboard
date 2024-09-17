# Game Payments Dashboard

This project contains a dashboard created in Tableau that visualizes factors affecting changes in revenue and the number of paying users. The data is extracted from a database containing information on in-game payments.

## Dashboard Title: Revenue, Customer, and Churn Analysis (Analiza Przychodów, Klientów i Wskaźników Churnu)

## Dashboard Contents

The dashboard includes the following charts:

1. **MRR & Paid Users ** - This chart displays monthly recurring revenue (MRR) along with the number of paying users over time.

   - **Bars:** Show the MRR value for each month, increasing from March to a peak in October, followed by a slight decrease in November.
   - **Line:** Represents the number of paying users, which grows over time and reaches a maximum of 189 users in October.
   - **Conclusion:** This chart illustrates an overall upward trend in both MRR and the number of paying users throughout the year.

2. **Customer Lifetime Value & Lifetime** - This chart provides an analysis of customer lifetime value (LTV) and the number of paid users across different months.

   - **Bars:** Represent the number of paying users in each month, with a visible decline after a peak in March.
   - **Line:** Shows the average customer lifetime (Lifetime) in each month, decreasing over the year.
   - **Conclusion:** This chart helps analyze how long customers remain active and their value over time.

3. **Revenue by Language** - This chart breaks down revenue by user language (en, ru, uk) and shows its changes over time.

   - **Bars:** Indicate the revenue for each language in each month. It is evident that English ("en") dominates in terms of revenue.
   - **Lines:** Represent revenue trends over time for different languages, with a notable increase, particularly from September to December.

4. **Revenue Change Factors ** - This waterfall chart visualizes the factors affecting revenue changes, such as new MRR, expansion, contraction, and churn.

   - **Bars:** Show the total revenue change in each month, clearly divided into different categories (colors) affecting the change.
   - **Line:** Illustrates the revenue change values, showing both increases and decreases throughout the year.

5. **Users Change Factors** - This chart shows the changes in the number of paying users, broken down by factors (e.g., new users, user churn).

   - **Bars:** Represent changes in the number of users, segmented by different statuses.
   - **Line:** Shows the overall change in the number of users each month.

6. **Churn Rate** - This line chart displays the churn rate (loss of customers) for revenue and the number of users.

   - **Lines:** The dark blue line represents the revenue churn rate, while the light blue line shows the user churn rate. There is an increase in churn in the second half of the year, with a peak in October (33.9% for revenue churn).

   - **Conclusion:** This chart allows analysis of how churn rate changes over time and its impact on the business.

## Overall Conclusion

This dashboard provides a comprehensive analysis of revenue dynamics, changes in the number of users, customer value, and churn rates throughout the year. Through various charts, it is possible to identify key factors influencing revenue growth and loss, aiding in strategic business decision-making.

## Data Extraction Code

The data for this dashboard is extracted using the following SQL code:

```sql
WITH payment_set AS (
  SELECT DISTINCT
    gp.user_id,
    gp.game_name,
    CAST(date_trunc('month', gp.payment_date) AS date) AS payment_month,
    CAST(date_trunc('month', gp.payment_date) - INTERVAL '1 month' AS date) AS payment_month_prev,
    CAST(date_trunc('month', gp.payment_date) + INTERVAL '1 month' AS date) AS payment_month_next,
    SUM(gp.revenue_amount_usd) AS revenue_amount,
    MIN(CAST(date_trunc('month', gp.payment_date) AS date)) OVER (PARTITION BY gp.user_id, gp.game_name) AS date_of_first_payment,
    MAX(CAST(date_trunc('month', gp.payment_date) AS date)) OVER (PARTITION BY gp.user_id, gp.game_name) AS date_of_last_payment
  FROM project.games_payments gp
  GROUP BY 1, 2, 3, 4, 5
),

payment_base_for_union AS (
  SELECT
    ps.user_id,
    ps.game_name,
    ps.payment_month,
    ps.revenue_amount,
    'revenue' AS status,
    ps.payment_month_prev,
    psp.revenue_amount AS ra_prev,
    CASE
      WHEN psp.revenue_amount IS NULL AND ps.payment_month = ps.date_of_first_payment THEN 'new MRR'
      WHEN psp.revenue_amount IS NULL AND ps.payment_month > ps.date_of_first_payment THEN 'Returning from CHURN'
      WHEN psp.revenue_amount < ps.revenue_amount THEN 'Expansion MRR'
      WHEN psp.revenue_amount > ps.revenue_amount THEN 'Contraction MRR'
    END AS prev_status,
    ps.revenue_amount - psp.revenue_amount AS change_value,
    ps.payment_month_next,
    psn.revenue_amount AS ra_next,
    CASE
      WHEN psn.revenue_amount IS NULL THEN 'CHURN'
    END AS next_status
  FROM payment_set ps
  LEFT JOIN payment_set psp ON psp.user_id = ps.user_id AND psp.game_name = ps.game_name AND psp.payment_month = ps.payment_month_prev
  LEFT JOIN payment_set psn ON psn.user_id = ps.user_id AND psn.game_name = ps.game_name AND psn.payment_month = ps.payment_month_next
),

payment_data AS (
  SELECT user_id, game_name, payment_month, revenue_amount, status
  FROM payment_base_for_union
  UNION
  SELECT user_id, game_name, payment_month, revenue_amount, prev_status
  FROM payment_base_for_union
  WHERE prev_status IN ('new MRR', 'Returning from CHURN')
  UNION
  SELECT user_id, game_name, payment_month, change_value, prev_status
  FROM payment_base_for_union
  WHERE prev_status IN ('Expansion MRR', 'Contraction MRR')
  UNION
  SELECT user_id, game_name, payment_month_next AS payment_month, -1 * revenue_amount, next_status
  FROM payment_base_for_union
  WHERE next_status = 'CHURN'
)

SELECT gpu.*, pd.payment_month, pd.revenue_amount, status
FROM project.games_paid_users gpu
LEFT JOIN payment_data pd ON pd.user_id = gpu.user_id AND pd.game_name = gpu.game_name
WHERE pd.payment_month NOT IN (SELECT MAX(payment_month) FROM payment_data);
```
