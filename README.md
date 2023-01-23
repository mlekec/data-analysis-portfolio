## About
I always loved to acquire new skills and knowledge, that is why after finishing my Bachelor's in "Multimedia and Internet Technologies" at Vytautas Magnus University I decided to go study Master's and currently I am studying in the program "Digital Culture" at Kaunas University of Technology. My main area of research during studies: how digital algorithms impact society and culture of algorithms. Beside formal studies, I am currently also studying Data Analytics at Turing College where I combine knowledge about algorithms and society with new skills in data analysis. Here I got the knowledge of using SQL, spreadsheets, Data Studio, all the main things for data gathering, analysis and visualization.

All the main skill that I aquired can be seen in my resume generated by Turing college: [Turing College resume](https://intra.turingcollege.com/s/mlekec-c35a7)

This is a repository to showcase skills, share projects and track my progress in Data Analytics field.

## Table of contents

- [About](#about)
- [Portfolio Projects](#portfolio-projects)
	+ [eCommerce clothing site customer analysis](#the-look-customer-analysis)
	+ [Olist marketplace analysis](#olist-marketplace-analysis)
	+ [AdwentureWorks Gender diversity analysis](#adwentureworks-gender-diversity-analysis)
	+ [AdventureWorks Mid year overview](#adventureworks-mid-year-overview)
- [Certificates and courses](#certificates-and-courses)
- [Contacts](#contacts)

## Portfolio projects

### The Look customer analysis
**Description:** The report is concentrating around customer analysis. Customer analytics refers to a collection of data points that indicate what customers are interacting with, how, and for how long. Interpreting those data can help understand what’s resonating among different customer segments. The report analyses the data from [TheLook](https://console.cloud.google.com/bigquery?p=bigquery-public-data&d=thelook_ecommerce&page=dataset) - eCommerce clothing site developed by the Looker team. This public dataset is hosted in Google BigQuery.

**Written report:** [Google Docs](https://docs.google.com/document/d/1mthjLkTk9takL971GuRUXddOIEkhm1g9z6Zrg5NsbYY/edit?usp=sharing)

**Dashboard:** [Data studio](https://datastudio.google.com/reporting/e23254eb-cf37-4150-ad61-5dac3078b029)

**Technology:** SQL (Bigquery), DataStudio

<details><summary>SQL code (click to expand)</summary>
<p>
For calculating the segments of customers (RFM):
  
```sql
   WITH --Compute for F & M
  frequency_monetary AS (
    SELECT
      user_id,
      MAX(created_at) AS last_purchase_date,
      COUNT(DISTINCT order_id) AS frequency,
      ROUND(SUM(sale_price), 2) AS monetary
    FROM `bigquery-public-data.thelook_ecommerce.order_items`
    WHERE created_at >= '2022-01-01' AND user_id IS NOT NULL
    GROUP BY user_id
  ),
--Compute for R
  recency_part AS (
    SELECT *,
      DATE_DIFF(DATE(reference_date), DATE(last_purchase_date), DAY) AS recency
    FROM (
      SELECT *, DATE(MAX(last_purchase_date) OVER ()) + 1 AS reference_date
      FROM frequency_monetary)
  ),
  percentiles_part AS (
    SELECT
      a.*,
      --All percentiles for MONETARY
      b.percentiles[offset(25)] AS m25,
      b.percentiles[offset(50)] AS m50,
      b.percentiles[offset(75)] AS m75,
      b.percentiles[offset(100)] AS m100,
      --All percentiles for FREQUENCY
      c.percentiles[offset(25)] AS f25,
      c.percentiles[offset(50)] AS f50,
      c.percentiles[offset(75)] AS f75,
      c.percentiles[offset(100)] AS f100,
      --All percentiles for RECENCY
      d.percentiles[offset(25)] AS r25,
      d.percentiles[offset(50)] AS r50,
      d.percentiles[offset(75)] AS r75,
      d.percentiles[offset(100)] AS r100
    FROM
      recency_part a,
      (SELECT APPROX_QUANTILES(monetary, 100) percentiles FROM recency_part) b,
      (SELECT APPROX_QUANTILES(frequency, 100) percentiles FROM recency_part) c,
      (SELECT APPROX_QUANTILES(recency, 100) percentiles FROM recency_part) d
  ),
  scoring_part AS (
    SELECT *,
      CAST(ROUND((f_score + m_score) / 2, 0) AS INT64) AS fm_score
    FROM (
      SELECT *,
        CASE WHEN monetary <= m25 THEN 1
          WHEN monetary <= m50 AND monetary > m25 THEN 2
          WHEN monetary <= m75 AND monetary > m50 THEN 3
          WHEN monetary <= m100 AND monetary > m75 THEN 4
        END AS m_score,
        CASE WHEN frequency <= f25 THEN 1
          WHEN frequency <= f50 AND frequency > f25 THEN 2
          WHEN frequency <= f75 AND frequency > f50 THEN 3
          WHEN frequency <= f100 AND frequency > f75 THEN 4
        END AS f_score,
        --Recency scoring is reversed
        CASE WHEN recency <= r25 THEN 4
          WHEN recency <= r50 AND recency > r25 THEN 3
          WHEN recency <= r75 AND recency > r50 THEN 2
          WHEN recency <= r100 AND recency > r75 THEN 1
        END AS r_score,
      FROM percentiles_part)
  ),
  naming_part AS (
    SELECT
      user_id,
      recency,
      frequency,
      monetary,
      r_score,
      f_score,
      m_score,
      fm_score,
      CONCAT(r_score, f_score, m_score) AS RFM_cell,
      (r_score*1+f_score*1+m_score*1) AS RFM_score,
      CASE WHEN (r_score = 4 AND fm_score = 4) OR (r_score = 4 AND fm_score = 3)
          THEN 'Champions' --Customers who bought most recently, most often and spend the most
        WHEN (r_score = 4 AND fm_score =2) OR (r_score = 3 AND fm_score = 3) OR (r_score = 2 AND fm_score = 4) OR (r_score = 2 AND fm_score = 3)
          THEN 'Loyal Customers' --Customers who bought most recently
        WHEN (fm_score = 4 AND r_score =3)
          THEN 'Big Spenders' --Customers who spend the most
        WHEN (r_score = 3 AND fm_score = 2) OR (r_score = 2 AND fm_score = 2) OR (r_score = 2 AND fm_score = 3)
          THEN 'Customers Needing Attention' --customers who do not purchase often and spend average amount
        WHEN (r_score = 3 AND fm_score = 1) OR (r_score = 4 AND fm_score = 1)
          THEN 'Promising' --customers who bought recently but did not spent a lot and are not frequent
        WHEN (r_score = 1 AND fm_score = 4) OR (r_score = 1 AND fm_score = 3)
          THEN 'At risk' --spend good amount but long time ago
        WHEN (r_score = 1 AND fm_score = 2) OR (r_score = 2 AND fm_score = 1)
          THEN 'Almost Lost' --Haven't purchased for some time, but purchased frequently and spend not a lot
        WHEN r_score = 1 AND fm_score = 1
          THEN 'Lost' --Haven't purchased for some time
      END AS rfm_segment
  FROM scoring_part
 )
SELECT *
FROM naming_part
ORDER BY user_id
```
  
  For finding average session duration and steps count:

```sql
WITH first_visit AS(
  SELECT
    DISTINCT(session_id),
    MAX(sequence_number) AS sequence_number,
    MIN(created_at) AS first_visit_date,
  FROM
    `bigquery-public-data.thelook_ecommerce.events`
  GROUP BY 1
),
last_event AS(
  SELECT
    session_id,
    first_visit.sequence_number AS sequence_number,
    first_visit_date AS first_visit_date,
    created_at AS last_event_date,
  FROM
    `bigquery-public-data.thelook_ecommerce.events`
  LEFT JOIN
    first_visit
  USING
    (session_id)
  WHERE
    created_at >= '2021-01-01' AND created_at<='2022-09-26'
),
difference_minutes AS(
  SELECT
    session_id,
    sequence_number,
    first_visit_date,
    last_event_date,
    TIMESTAMP_DIFF(last_event_date, first_visit_date, MINUTE) AS difference_in_minutes
  FROM
    last_event
  WHERE
    DATE_TRUNC(first_visit_date, DAY) = DATE_TRUNC(last_event_date, DAY)
  ORDER BY
    first_visit_date
)
SELECT
  DATE_TRUNC(first_visit_date, DAY) AS first_visit_purchase_date,
  AVG(sequence_number) AS avg_num_steps, ROUND(AVG(difference_in_minutes), 2) AS average_difference_minutes
FROM
  difference_minutes
GROUP BY
  first_visit_purchase_date, sequence_number
ORDER BY
  first_visit_purchase_date
```  

</p>
</details>
  
### Olist marketplace analysis
**Description:** The report is concentrating around delivery time analysis, customers and sellers analysis and geographical pricing analysis. The report analyses the data from [Olist](https://olist.com/pt-br/) -  Brazilian marketplace that operates in the e-commerce segment, but is not an e-commerce service itself. It operates as a SaaS (Software as a Service) technology company since 2015.. Database is consisting of transactional payments and orders data from Olist of 100k orders from 2016 to 2018. 

**Dashboard:** [Data studio](https://datastudio.google.com/reporting/f2226f64-4dac-4fa1-967e-07057613c1f4)
  
**Presentation:** [Google Slides](https://docs.google.com/presentation/d/1mJWphKeIyq-TpfDpjzvgzIFC58z_NHwoD_cDzjdjMPY/edit?usp=sharing)

**Technology:** SQL (Bigquery), DataStudio
  
<details><summary>SQL code (click to expand)</summary>
<p>
  
  For finding order value per customer city and state:

```sql
   SELECT
    customer_state,
    customer_city,
    SUM(price)/COUNT(order_id) AS order_value
  FROM
    `tc-da-1.olist_db.olist_order_items_dataset` items
  INNER JOIN
    `tc-da-1.olist_db.olist_orders_dataset` orders USING (order_id)
  INNER JOIN
    `olist_db.olist_customesr_dataset` customers USING (customer_id)
  GROUP BY customer_state, customer_city
```
  
  For finding revenue per customer city and state:
  
  ```sql
   SELECT
    customer_state,
    customer_city,
    SUM(price + freight_value) AS revenue
  FROM
    `tc-da-1.olist_db.olist_order_items_dataset` items
  INNER JOIN
    `tc-da-1.olist_db.olist_orders_dataset` orders USING (order_id)
  INNER JOIN
    `olist_db.olist_customesr_dataset` customers USING (customer_id)
  GROUP BY customer_state, customer_city
```
</p>
</details>

### AdwentureWorks Gender diversity analysis
**Description:** The report is concentrating around gender diversity analysis. Diversity metrics are the measurable numerical values that help HR to see workforce demographics and assess the efforts of the company towards inclusive practices. Like other metrics for HR, diversity metrics allow getting insight into a company's workforce and the overall health of the company. And diversity metrics can help to improve Diversity and inclusion practices. The report analyses the data from [AdwentureWorks](https://i0.wp.com/improveandrepeat.com/wp-content/uploads/2018/12/AdvWorksOLTPSchemaVisio.png?ssl=1). In the database there is information about sales, purchasing, personel, production, orders, human recources. 

**Dashboard:** [Data studio](https://datastudio.google.com/reporting/9a8aadac-d7c4-4efb-80ec-0ecf2e216a44)
  
**Presentation:** [Google Slides](https://docs.google.com/presentation/d/1RVRkHpKqUcr-HiB8u_z2BtyBBtY1XO25fOLANZPJa2k/edit?usp=sharing)

**Technology:** SQL (Bigquery), DataStudio

<details><summary>SQL code (click to expand)</summary>
<p>

  For finding pay rate by job title and gender:
  
```sql
   SELECT
    EmployeeID, Gender, MAX(Rate) AS pay_rate, department.Name AS department_name, Title
   FROM
    `tc-da-1.adwentureworks_db.employeedepartmenthistory`
   LEFT JOIN
    `adwentureworks_db.employee` USING (EmployeeId)
   LEFT JOIN
    `adwentureworks_db.employeepayhistory` USING (EmployeeId)
   LEFT JOIN
    `adwentureworks_db.department` department USING (DepartmentId)
   GROUP BY
    EmployeeID, Gender, department_name, Title
   ORDER BY 
    EmployeeID
```

</p>
</details>
  
### AdventureWorks Mid year overview
**Description:** The report is concentrating around main metrics during mid-yer overview. The midpoint is a great time to check if the business is on track and what corrective actions might be needed to take to achieve or exceed the company's goals. The report analyses the data from [AdwentureWorks](https://i0.wp.com/improveandrepeat.com/wp-content/uploads/2018/12/AdvWorksOLTPSchemaVisio.png?ssl=1). In the database there is information about sales, purchasing, personel, production, orders, human recources. 

**Dashboard:** [Data studio](https://datastudio.google.com/reporting/cf7741bb-809d-4019-b2cb-e24c2eb79db3)
  
**Presentation:** [Google Slides](https://docs.google.com/presentation/d/19wFm-KgB6wwCpUTA08eVGewUx1R5I0ErMtzqi3fKFnw/edit?usp=sharing)

**Technology:** SQL (Bigquery), DataStudio
  
<details><summary>SQL code (click to expand)/age</summary>
<p>

  For finding customers and revenue by region:
  
```sql
  SELECT
    salesOrders.OrderDate AS orderDate,
    territory.CountryRegionCode,
    territory.Name AS Region,
    COUNT(salesOrders.SalesOrderID) AS NumberOfOrders,
    COUNT(DISTINCT salesOrders.CustomerID) AS NumberOfCustomers,
    COUNT(DISTINCT salesOrders.SalesPersonID) AS NumberOfSalesPerson,
    ROUND(SUM(salesOrders.TotalDue), 2) AS TotalAmount,
  FROM
    `tc-da-1.adwentureworks_db.salesorderheader` AS salesOrders
  LEFT JOIN
    `tc-da-1.adwentureworks_db.salesterritory` AS territory
  ON
    salesOrders.TerritoryID = territory.TerritoryID
  GROUP BY
    orderDate, territory.CountryRegionCode, territory.Name
```

</p>
</details>

## Certificates and courses

  -[Microsoft Power BI Data Analyst](https://www.udemy.com/certificate/UC-c5f81b39-5705-45b5-9124-80b8c36950e5/) (Jan 2023) (Udemy)
  
  -[Advanced SQL](https://www.kaggle.com/learn/certification/monikalekeckait/advanced-sql) (Nov 2022) (Kaggle)
  
  -[Data Visualization](https://www.kaggle.com/learn/certification/monikalekeckait/data-visualization) (Nov 2022) (Kaggle)
  
  -[Pandas](https://www.kaggle.com/learn/certification/monikalekeckait/pandas) (Nov 2022) (Kaggle)
  
  -[Data Analytics](https://intra.turingcollege.com/s/mlekec-c35a7) (Oct 2022) (Turing College)
  
  -[Python](https://www.kaggle.com/learn/certification/monikalekeckait/python) (Oct 2022) (Kaggle)

  -[Introduction to Data Studio](https://analytics.google.com/analytics/academy/certificate/0fEq98a4QYmVfYdeksO1Zg) (Jul 2022) (Google Analytics)

  -[Advanced Google Analytics](https://www.linkedin.com/in/monika-lekeckaite/) (Apr 2022) (LinkedIn)

  -[Fundimentals of Digital Marketing](https://www.linkedin.com/in/monika-lekeckaite/) (Sep 2019) (The Open University)

  -[Elements of AI](https://certificates.mooc.fi/validate/p1cxbbu3ry) (Mar 2019) (University of Helsinki)
  
  
## Contacts
LinkedIn: [/monika-lekeckaite](https://www.linkedin.com/in/monika-lekeckaite/)

E-mail: monika.lekeckaite@gmail.com
