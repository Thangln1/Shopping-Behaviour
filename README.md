-- Here I will explore the data
-- Prior to this I checked the schema. The data types are approrpiate for each column
SELECT 
  *
FROM
  `shoppingbehavior.sb.sbb`
;
  -- Identify the most popular product and category by Season
  -- Here we see the top 5 Items are Jacket (fall), sunglasses (winter), sweater (spring), Pants (winter), shirt (winter)
  -- We also see popular categories in each season outwear (fall), accessories (winter), clothing, spring, winter
  -- Upon reviewing the code and data, there may be inaccurate results in regards to total of categories. I will run another code to check
SELECT 
  ItemPurchased,
  COUNT (ItemPurchased) AS Item_Count,
  Category,
  Count (Category) AS Category_Count,
  Season,
  FROM
  `shoppingbehavior.sb.sbb`
  GROUP BY 
    Season, ItemPurchased, Category
ORDER BY
Item_Count DESC;

  -- Here I will check the count of each category and analyse the relationship with season
  -- The code below shows the correct category count and its relationship with season
  -- Upon running the code, We have identified Clothing as the most popular in all Season. The least popular is outwear in all seasons
SELECT 
  Category,
  Count (Category) AS Category_Count,
  Season,
  FROM
  `shoppingbehavior.sb.sbb`
  GROUP BY 
    Season, Category
  ORDER BY 
    Category_Count DESC;

    -- Now I will identify which category brought in the most revenue in each season
    -- Here we see clothing in spring, accessories in fall, footwear in spring, outwear in fall

SELECT 
  Category,
  Count (Category) AS Category_Count,
  Sum (PurchaseUSD) As Revenue,
  Season,
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY 
    Season, Category
ORDER BY 
    Revenue DESC;

-- Season revenue. Here we see the most revenue was generated in fall, followed by spring, winter, and summer.
SELECT 
  SUM (PurchaseUSD) As Revenue,
  Season,
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
  Season
ORDER BY 
Revenue DESC;

-- relationship between shipping type and season and PurchaseUSD
SELECT 
 ShippingType,
 COUNT (ShippingType) AS ShippingType_Count,
 SUM (PurchaseUSD) AS Revenue,
 Season
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
  ShippingType, Season;

  -- Percentage of people using shipping types in each season
  -- Below I created a subquery within the select statement of the main query to allow me to calculator percentage of shoppers using different shipping type methods each season
  -- I have created a temporary table to analyse
--Main query
CREATE TABLE `shoppingbehavior.sb.sbb_customer_revenue`
OPTIONS(
  expiration_timestamp=TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 3 DAY)
) AS
SELECT
 ShippingType,
 COUNT (ShippingType) AS ShippingType_Count,
 SUM (PurchaseUSD) AS Revenue
 --subquery
(SELECT COUNT(CustomerID) AS CustomerID_Count FROM `shoppingbehavior.sb.sbb`) AS CustomerCount
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
  ShippingType;
--Here I have worked out shipping type use percentage which is only ranges by ~0.8%
-- I will now check check correlation with season
-- 
SELECT
  *,
  (ShippingType_Count / CustomerCount) * 100 AS Percentage_of_Customers
FROM
  `shoppingbehavior.sb.sbb_customer_revenue`;

-- New temp table to include season
CREATE TABLE `shoppingbehavior.sb.sbb_customer_revenue2`
AS
SELECT
 ShippingType,
 COUNT (ShippingType) AS ShippingType_Count,
 SUM (PurchaseUSD) AS Revenue,
 Season,
 --subquery
(SELECT COUNT(CustomerID) AS CustomerID_Count FROM `shoppingbehavior.sb.sbb`) AS CustomerCount
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
  ShippingType, Season;

-- Now I have created a temp table. I will analyse percentage usage of shipping types across different seasons
-- This gives us percentage of customers preference of shipping type THROUGHOUT the year. However, for a more accurate representation of shopping behaviour in each season. I will need to count how many purchases were made in each Season then analyse to rank shippingtype %.
SELECT
ShippingType,
Season,
Revenue,
(ShippingType_Count / CustomerCount) * 100 AS Percentage
FROM 
`shoppingbehavior.sb.sbb_customer_revenue2`
ORDER BY
Season DESC, Percentage DESC;

-- 1 create new query based on customers in each season. From the data we know each customer purchases in ONLY 1 season, not more. 
-- I now have results from the query. This will be the subquery (2nd table) which I will inner join with the above table by Season
SELECT
  COUNT (CustomerID) AS Customers,
  Season,
FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
Season;

-- creating a query using left inner join using Season (common key)
-- I have completed but now I have duplicate seasons. I will select all data from first table (c) then customers from second table (o)
-- I then created a table for below for me to analyse later on
CREATE TABLE `shoppingbehavior.sb.sbb_customer_revenue3`
 AS
Select
  c.*,
  o.Customers
  FROM 
`shoppingbehavior.sb.sbb_customer_revenue2` c
LEFT JOIN
  (SELECT
    COUNT (CustomerID) AS Customers,
    Season,
  FROM
    `shoppingbehavior.sb.sbb`
  GROUP BY
    Season) o
ON c.Season = o.Season;

-- I will analyse temp table3 to see if there is a relationship between shippingtype and season
CREATE TABLE `shoppingbehavior.sb.shipping_perc_season` AS
SELECT  
  ShippingType,
  Season,
  (ShippingType_Count/Customers)*100 AS ShipType_Percent
FROM
  `shoppingbehavior.sb.sbb_customer_revenue3`
ORDER BY
  Season ASC, ShipType_Percent DESC;

-- I will combined the above two queries to join shiptype percentage across easons to the first table and left joining shipping_perc_season query
SELECT
s.ShippingType,
s.Season,
s.Revenue,
p.ShipType_Percent
FROM
`shoppingbehavior.sb.sbb_customer_revenue3` s
LEFT JOIN
`shoppingbehavior.sb.shipping_perc_season` p
ON s.ShippingType = p.ShippingType AND s.Season = p.Season
GROUP BY
  ShippingType, Season, Revenue, ShipType_Percent;

  -- Relationship between gender and buying frequency
  -- We see both genders shopped more frequently every quarterly with previous purchases being significantly highest in the quarterly group
SELECT
  Gender,
  COUNT (CustomerID) AS Customers,
  FrequencyofPurchases,
  COUNT (FrequencyofPurchases) AS Frequencypurchase_count,
  SUM (PreviousPurchases) AS PreviousPurchase_sum
FROM
   `shoppingbehavior.sb.sbb`
GROUP BY
  Gender, FrequencyofPurchases
ORDER BY
  PreviousPurchase_sum DESC;

-- Relationship between Location and Size and PurchaseUSD (revenue). This will require multiple CTEs as I want to create percentage of sizes in each state and Revenue generated by each size

-- 1st CTE
WITH CTE_customers AS
(
  SELECT
    Location,
    COUNT (CustomerID) AS Customers
  FROM
    `shoppingbehavior.sb.sbb`
  GROUP BY
    Location
)
--2nd CTE
, CTE_Size AS
(
SELECT
  Location,
  Size,
  COUNT (Size) AS Size_Total
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
  Location, Size
)
--3rd CTE. This will look at revenue generated by each size in each state
, CTE_Revenue AS
(
SELECT
 Location,
  Size,
  SUM (PurchaseUSD) AS Revenue_USD
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
Location, Size
)
-- main query combining to create customer percentage and revenue according to sizes across states 
SELECT
c.Location,
s.Size,
(s.Size_Total/c.Customers)*100 AS Percentage,
r.Revenue_USD
FROM
  CTE_Size s
LEFT JOIN
  CTE_customers c
ON s.Location = c.Location
LEFT JOIN
 CTE_Revenue r
ON s.Location = r.Location AND s.Size = r.Size -- by adding these two add ONs I no longer got duplicate columns for locations
GROUP BY
Location, Percentage, Size, Revenue_USD
ORDER BY 
  Revenue_USD DESC;

-- Create SumPurchase By size and season for geographic visualisation purposes

WITH CTE_customers AS
(
  SELECT
    Location,
    COUNT (CustomerID) AS Customers
  FROM
    `shoppingbehavior.sb.sbb`
  GROUP BY
    Location
)
--2nd CTE
, CTE_Size AS
(
SELECT
  Location,
  Size,
  COUNT (Size) AS Size_Total
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
  Location, Size
)
--3rd CTE. This will look at revenue generated by each size in each state
, CTE_Revenue AS
(
  SELECT
    Location,
    Season,
    Size,
    SUM (PurchaseUSD) AS Revenue_USD
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
Location, Size, Season
)
-- main query combining to create customer percentage and revenue according to sizes across states 
SELECT
c.Location,
s.Size,
(s.Size_Total/c.Customers)*100 AS Percentage,
r.Revenue_USD,
r.Season
FROM
  CTE_Size s
LEFT JOIN
  CTE_customers c
ON s.Location = c.Location
LEFT JOIN
 CTE_Revenue r
ON s.Location = r.Location AND s.Size = r.Size
GROUP BY
Location, Percentage, Size, Revenue_USD, Season
ORDER BY 
  Revenue_USD DESC;

-- Relationship between male and female subscribers, non-subscribers and buying frequencies
SELECT
SUM(CASE Gender WHEN 'Male' THEN 1 ELSE 0 END) AS Male,
SUM(CASE Gender WHEN 'Female' THEN 1 ELSE 0 END) AS Female,
FrequencyofPurchases,
SubscriptionStatus,
PromoCodeUsed,
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
  FrequencyofPurchases, SubscriptionStatus, PromoCodeUsed;

-- I will correlate previous purchases with subscriptions and promocode used, then inner join with above table
WITH CTE_PP_Sub_PC_FC AS
(
SELECT
SubscriptionStatus,
PromoCodeUsed,
FrequencyofPurchases,
SUM (PreviousPurchases) AS Total_PreviousPurchases
FROM
  `shoppingbehavior.sb.sbb` 
GROUP BY
  SubscriptionStatus, PromoCodeUsed, FrequencyofPurchases
)
SELECT
SUM(CASE Gender WHEN 'Male' THEN 1 ELSE 0 END) AS Male,
SUM(CASE Gender WHEN 'Female' THEN 1 ELSE 0 END) AS Female,
c.FrequencyofPurchases,
c.SubscriptionStatus,
c.PromoCodeUsed,
o.Total_PreviousPurchases
FROM
  `shoppingbehavior.sb.sbb` c
LEFT JOIN
  CTE_PP_Sub_PC_FC o
  ON c.SubscriptionStatus = o.SubscriptionStatus AND c.PromoCodeUsed = o.PromoCodeUsed AND c.FrequencyofPurchases = o.FrequencyofPurchases
GROUP BY
 FrequencyofPurchases, SubscriptionStatus, PromoCodeUsed, o.Total_PreviousPurchases;

-- Gender_Percentage of subscription status and promocode used by Season and Location
WITH CTE_GenderCount AS
(
SELECT
  Gender,
  Count (Gender) AS Gender_Total,
  SUM (PreviousPurchases) AS PreviousPurchase_Sum,
  AVG (PreviousPurchases) AS PreviousPurchase_Avg,
  SubscriptionStatus,
  PromoCodeUsed,
 FROM
  `shoppingbehavior.sb.sbb`
GROUP BY
  Gender, SubscriptionStatus, PromoCodeUsed
)
, CTE_GenderTotal AS
(
  SELECT
    Gender,
    Count (Gender) AS Gender_Total2
  FROM
    `shoppingbehavior.sb.sbb`
  GROUP BY
    Gender
)
  SELECT
    o.Gender,
    o.Gender_Total,
    o.PreviousPurchase_Sum,
    o.PreviousPurchase_Avg,
    o.SubscriptionStatus,
    o.PromoCodeUsed,
    (o.Gender_Total/p.Gender_Total2)*100 AS Gender_Percentage
  FROM
    CTE_GenderCount o
  LEFT JOIN CTE_GenderTotal p
    ON o.Gender = p.Gender
  GROUP BY
    o.Gender, o.PromoCodeUsed, o.SubscriptionStatus,o.Gender_Total, o.PreviousPurchase_Sum, o.PreviousPurchase_Avg, Gender_Percentage
