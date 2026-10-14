# Bicycle-Manufacturer-Analytics
## I. Introduction

This project uses SQL and Google BigQuery to analyze the AdventureWorks2019 dataset, focusing on sales performance, product demand, customer retention, inventory management, and purchasing activities. By applying SQL techniques such as joins, common table expressions (CTEs), window functions, and cohort analysis, the project aims to uncover business trends and provide data-driven insights to support decision-making.

## II. Objective
- Analyze sales performance and product growth.
- Identify top-performing sales territories.
- Evaluate customer retention and repeat purchases.
- Monitor inventory levels and stock-to-sales ratios.
- Assess seasonal discount costs.
- Track pending purchase orders and outstanding values.

## III. Tools Used
- SQL (Google BigQuery)
- AdventureWorks2019 Dataset

## IV. Data Access
- Log in to your **Google Cloud Platform** account and create a new project.
- Navigate to the **BigQuery console** and select your newly created project.
- In the navigation panel, select **"Add Data"** and then **"Star a project by name"**.
- Enter the project name **"adventurework2019"**.

## V. Analysis Journey
### Query 1: Calc Quantity of items, Sales value & Order quantity by each Subcategory in L12M
``` sql
SELECT FORMAT_DATE("%b %Y", a.ModifiedDate) period
  , c.Name
  , sum(OrderQty) qty_item
  , sum(LineTotal) total_sales
  , count(distinct SalesOrderID) order_cnt
FROM `adventureworks2019.Sales.SalesOrderDetail` a
left join `adventureworks2019.Production.Product` b
  using(ProductID)
left join `adventureworks2019.Production.ProductSubcategory` c
  on cast(b.ProductSubcategoryID as INT) = c.ProductSubcategoryID
where date(a.ModifiedDate) >= (SELECT DATE_SUB(MAX(DATE(ModifiedDate)), INTERVAL 12 MONTH)
                               FROM `adventureworks2019.Sales.SalesOrderDetail`)
group by period, c.Name
order by c.Name;
```

#### Result:
<img width="563" height="675" alt="Image" src="https://github.com/user-attachments/assets/fa02a8a6-b140-4c6c-b06c-e6d3736329e8" />

#### Insights:
- Bike Racks recorded strong sales, peaking at 492 items and approximately 35,589 in March 2014.
- Bike Stands showed consistent monthly demand, with quantities ranging from 2 to 31 items across the displayed period.
- Sales performance varied considerably by subcategory, suggesting different demand levels and revenue contributions across product groups.

### Query 2: Calc % YoY growth rate by SubCategory & release top 3 cat with highest grow rate. (qty_diff = qty_item / prv_qty - 1)
``` sql 
with raw_data as (
  SELECT FORMAT_DATE("%Y", a.ModifiedDate) year
    , c.Name
    , sum(a.OrderQty) qty_item
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Production.Product` b
    using(ProductID)
  left join `adventureworks2019.Production.ProductSubcategory` c
    on cast(b.ProductSubcategoryID as INT) = c.ProductSubcategoryID
  group by year, c.Name
  order by year
)
, previous as (
  SELECT *
    , lead(qty_item) over(partition by name order by year desc) prv_qty
  FROM raw_data
)
, diff as (
  SELECT *
    , round(qty_item / prv_qty - 1, 2) qty_diff
  FROM previous
  where prv_qty is not NULL
  order by qty_diff desc
)
, ranking as (
  select * 
    , dense_rank() over(order by qty_diff desc) rk
  from diff
)

select name
  , qty_item
  , prv_qty
  , qty_diff
from ranking
where rk <= 3;
```

#### Result:
<img width="635" height="108" alt="Image" src="https://github.com/user-attachments/assets/4a8bc36b-2ad4-41bd-a6b7-a848bf4c9033" />

#### Insights:
- Mountain Frames recorded the highest growth, with item quantity increasing from 510 to 3,168, equivalent to approximately 521% growth.
- Socks ranked second, increasing from 523 to 2,724 items, or approximately 421% growth.
- Road Frames ranked third, increasing from 1,137 to 5,564 items, or approximately 389% growth.
-These subcategories experienced substantial increases in item quantities and may warrant further investigation into the factors driving their growth.

### Query 3: Ranking Top 3 TeritoryID with biggest Order quantity of every year. (If there's TerritoryID with same quantity in a year, do not skip the rank number)
``` sql
with raw_data as (
  SELECT FORMAT_DATE("%Y", a.ModifiedDate) yr
    , c.TerritoryID
    , sum(a.OrderQty) order_cnt
  FROM `adventureworks2019.Sales.SalesOrderDetail` a
  left join `adventureworks2019.Sales.SalesOrderHeader` b
    using(SalesOrderID)
  left join `adventureworks2019.Sales.Customer` c
    using(CustomerID)
  group by yr, c.TerritoryID
)
, ranking as (
  SELECT *
    , dense_rank() over(partition by yr order by order_cnt desc) rk
  FROM raw_data
  order by yr desc
)

SELECT *
FROM ranking
where rk <= 3;
```

#### Result:
<img width="484" height="351" alt="Image" src="https://github.com/user-attachments/assets/c14483b5-4b1f-47fa-8b88-1053cfbe121e" />

#### Insights:
- Territory 4 ranked first in every year from 2011 to 2014, with order quantities peaking at 26,682 in 2013.
- Territory 6 consistently ranked second, while Territory 1 ranked third throughout the period.
- All three territories recorded their highest order quantities in 2013, followed by lower quantities in 2014.
- The consistent rankings suggest that these territories were important contributors to sales volume during the analyzed period.

### Query 4: Calc Total Discount Cost belongs to Seasonal Discount for each SubCategory.(Discount Cost = Disct Pct * Unit Price * Item Qty)
``` sql
SELECT FORMAT_DATE("%Y", a.ModifiedDate) year
  , c.Name
  , sum(d.DiscountPct * a.UnitPrice * a.OrderQty) total_cost
FROM `adventureworks2019.Sales.SalesOrderDetail` a
left join `adventureworks2019.Production.Product` b
  using(ProductID)
left join `adventureworks2019.Production.ProductSubcategory` c
  on cast(b.ProductSubcategoryID as INT) = c.ProductSubcategoryID
left join `adventureworks2019.Sales.SpecialOffer` d
   on a.SpecialOfferID = d.SpecialOfferID
where lower(Type) like '%seasonal discount%' 
group by year, c.Name;
```

#### Result:
<img width="480" height="80" alt="Image" src="https://github.com/user-attachments/assets/b1b8ce9c-b997-461e-9a9b-b80545175bc3" />

#### Insights:
- Seasonal discounts were recorded for Helmets in 2012 and 2013.
- Discount costs increased from approximately 827.65 in 2012 to 1,606.04 in 2013, an increase of around 94%.
- The higher discount cost may reflect increased promotional activity or a change in discounted sales volume. Further analysis is needed to determine whether the discounts generated sufficient additional sales.

### Query 5: Retention rate of Customer in 2014 with status of Successfully Shipped (Cohort Analysis)
``` sql
with info as (
  SELECT extract(month FROM ModifiedDate) month_order
    , extract(year FROM ModifiedDate) year
    , CustomerID
    , count(distinct SalesOrderID) sales_total
  FROM `adventureworks2019.Sales.SalesOrderHeader` 
  where Status = 5 and extract(year FROM ModifiedDate) = 2014
  group by 1,2,3
)
, row_n as (
  select *
    , row_number() over(partition by CustomerID order by month_order) row_num
  from info
)
, first_order as (
  select distinct month_order as month_join, year, CustomerID
  from row_n
  where row_num = 1
)
, all_join as (
  select distinct a. CustomerID
    , a.month_order
    , a.year
    , b.month_join
    , concat('M', a.month_order - b.month_join) as month_diff
  from info a
  left join first_order b
    using(CustomerID)
  order by CustomerID
)

select distinct month_join, month_diff
  , count(distinct CustomerID) customer_cnt
from all_join
group by 1,2
order by 1,2;
```

#### Result:
<img width="511" height="675" alt="Image" src="https://github.com/user-attachments/assets/22a41053-b722-411e-9389-987bf8d54668" />

#### Insights:
- January had the largest initial customer cohort, with 2,076 customers, followed by March with 1,918 and April with 1,906.
- The January cohort recorded 252 customers in M3, compared with 78 in M1 and 89 in M2, indicating that repeat activity varied across subsequent months.
- The results show a decline in observed customer counts across later cohort periods, although some intermediate months experienced increases.
- Customer retention should be evaluated as a percentage of each original cohort rather than by comparing customer counts alone.

### Query 6: Trend of Stock level & MoM diff % by all product in 2011. If %gr rate is null then 0. Round to 1 decimal
``` sql
with raw_data as (
  SELECT b.Name
    , extract(month FROM c.ModifiedDate) mth
    , extract(year FROM c.ModifiedDate) yr
    , sum(c.StockedQty) stock_qty
  FROM `adventureworks2019.Production.Product` b
  left join `adventureworks2019.Production.WorkOrder` c
    using (ProductID)
  where extract(year FROM c.ModifiedDate) = 2011
  group by 1,2,3
  having stock_qty is not NULL
  order by 1
)
, stock_previous as (
  select *
    , lead(stock_qty) over(partition by Name order by mth desc) stock_prv
  from raw_data
)

select *
  , round(case when stock_prv is null then 0
              else (stock_qty / stock_prv - 1) * 100 end, 1) diff
from stock_previous;
```

#### Result:
<img width="644" height="675" alt="Image" src="https://github.com/user-attachments/assets/c050eec0-c95c-4c47-8bf0-87e8dc6b04d4" />

#### Insights:
- Stocked quantities fluctuated substantially across the displayed products, indicating changes in production or inventory replenishment throughout the year.
- For BB Ball Bearing, stock increased by 116.8% in October and 144.1% in July, followed by declines of 24.2% in November and 41.7% in December.
- Blade showed a similar pattern, increasing by 120.1% in October before declining by 48.8% in December.
- These fluctuations highlight the importance of monitoring inventory movements to identify potential supply imbalances.

### Query 7: Calc Ratio of Stock / Sales in 2011 by product name, by month. (Order by month desc, ratio desc. Round Ratio to 1 decimal)
``` sql
with sale_info as (
  select extract(month FROM a.ModifiedDate) mth
      , extract(year FROM a.ModifiedDate) yr
      , a.ProductID
      , b.Name
      , sum(OrderQty) sales
  from `adventureworks2019.Sales.SalesOrderDetail` a 
  left join `adventureworks2019.Production.Product` b 
    using(ProductID)
  where extract(year FROM a.ModifiedDate) = 2011
  group by 1,2,3,4
  order by mth desc, sales
  )
, stock_info as (
  select extract(month FROM c.ModifiedDate) mth
    , extract(year FROM c.ModifiedDate) yr
    , b.Name
    , sum(StockedQty) stock
  from `adventureworks2019.Production.WorkOrder` c
  left join `adventureworks2019.Production.Product` b
    using (ProductID)
  where extract(year FROM c.ModifiedDate) = 2011
  group by 1,2,3
  order by mth desc, stock
)

select d.mth
  , d.yr
  , d.ProductID
  , e.Name
  , COALESCE(d.sales, 0) sales
  , COALESCE(e.stock, 0) stock
  , round(COALESCE(e.stock, 0) / COALESCE(d.sales, 0), 1) ratio
from sale_info d
join stock_info e
  on d.Name = e.Name
    and d.mth = e.mth
order by d.mth desc, ratio desc;
```

#### Result:
<img width="794" height="674" alt="Image" src="https://github.com/user-attachments/assets/b8ea259d-b260-4e73-a48c-1fcc93a5a6ce" />

#### Insights:
- In December, HL Mountain Frame – Black, 48 had a stock-to-sales ratio of 27.0, with 27 units stocked against one unit sold.
- HL Mountain Frame – Black, 42 followed with a ratio of 26.0, while HL Mountain Frame – Silver, 38 recorded 16.0.
- Several road and mountain frame products had ratios close to 1.0, indicating that recorded stocked quantities were approximately equal to sales quantities for that month.
- High ratios may indicate potential overstocking or weak demand, while low ratios require monitoring to assess possible stock shortages.

### Query 8: No of order and value at Pending status in 2014
``` sql
SELECT extract(year FROM ModifiedDate) yr
  , Status 
  , count(distinct PurchaseOrderID) order_Cnt
  , sum(TotalDue) value
FROM `adventureworks2019.Purchasing.PurchaseOrderHeader`
where extract(year FROM ModifiedDate) = 2014
  and Status = 1
group by 1, 2;
```

#### Result: 
<img width="459" height="53" alt="Image" src="https://github.com/user-attachments/assets/2beaf3b1-479b-4641-ab9e-421129db5bfe" />

#### Insights:
- There were 224 pending purchase orders in 2014, with a combined value of approximately 3.87 million.
- The outstanding value represents a substantial amount of purchasing commitments that had not reached a completed status in the analyzed data.
- Monitoring these orders can help purchasing and finance teams track outstanding commitments, assess supplier delays, and improve cash-flow planning.

## VI. Conclusions & Recommendations
### 1. Conclusions
- Sales performance: Product demand varied across subcategories, while Mountain Frames, Socks, and Road Frames recorded the strongest year-over-year quantity growth.
- Regional contribution: Territories 4, 6, and 1 consistently ranked among the top three territories by order quantity.
- Customer retention: Repeat purchasing activity varied by cohort, highlighting the need to monitor retention rates over time.
- Inventory management: Large stock fluctuations and high stock-to-sales ratios may indicate opportunities to improve production planning and inventory control.
- Financial monitoring: Seasonal discount costs increased in 2013, while pending purchase orders represented approximately 3.87 million in outstanding purchasing value.

### 2. Recommendations
- Optimize product strategy: Investigate the drivers of growth in Mountain Frames, Socks, and Road Frames, and align inventory and marketing plans with demand.
- Strengthen territory performance: Analyze the sales drivers in Territories 4, 6, and 1, while investigating the decline in order quantities in 2014.
- Improve customer retention: Track cohort retention percentages and develop targeted follow-up campaigns to encourage repeat purchases.
- Enhance inventory control: Review products with high stock-to-sales ratios and use historical demand to improve replenishment and production decisions.
- Evaluate promotional effectiveness: Compare incremental sales and gross profit against seasonal discount costs before expanding promotional activities.
- Monitor purchasing commitments: Track pending purchase order values, order aging, and expected delivery dates to improve purchasing visibility and cash-flow management.




