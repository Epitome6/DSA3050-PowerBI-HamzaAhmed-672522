# DSA3050-PowerBI-HamzaAhmed-672522
### Dataset

1. Source : [https://www.kaggle.com/datasets/apoorvaappz/global-super-store-dataset?select=Global\_Superstore2.xlsx](https://www.kaggle.com/datasets/apoorvaappz/global-super-store-dataset?select=Global_Superstore2.xlsx)
2. This dataset consists of data on online shopping that took place worldwide. It has both product and customer end details. It is in excel format. It has 24 columns and 51,291 rows
3. I selected it because is a suitable dataset for customer and product analysis along with sales performance and revenue analysis.
4. The main variables are : Sales, Profit, Shipping Cost, Category, Order Priority

5. Global Superstore is not consistently profitable at the transaction level, and there is currently no clear visibility into where losses are concentrated or what is driving them. This BI solution is designed to move beyond reporting what was sold, toward explaining where profitability is strong versus weak, and why. Finding the specific combinations of market, category, segment, and discount behavior that most need management's attention.
6. Analytical questions my Power BI solution should answer

   1. What is the overall trend in Sales, Profit, and Profit Margin from 2011–2014, and how does year-over-year performance compare?
   2. Which markets, regions, and countries generate the strongest sales ;and are any of them generating strong sales but weak or negative profit? 
   3. Which product categories and sub-categories are most and least profitable, and does that ranking match how much revenue they actually generate?
   4. Is there a measurable relationship between discount level and profitability ;do higher-discount transactions account for the percentage of orders that break even or lose money?
   5. How does profitability differ across customer segments (Consumer, Corporate, Home Office), and which segment delivers the most value per customer?
   6. Does 'Ship Mode' or 'Order Priority' affect shipping cost and fulfillment time, and are faster shipping options quietly eroding margin through higher shipping costs?

### Power Query

1. Convert Order Date and Ship Date to a real Date type. All other columns were detected correctly

   * Problem: Order Date and Ship Date are imported as text strings in DD-MM-YYYY format (e.g. "01-01-2011"), not native dates. Left as text, they can't be used in date slicers, sorted correctly, or related to a Date table.
   * Transformation: Selected and right clicked both columns → Change Type → Using Locale... → Date;English(Kenya), explicitly confirming the locale as Day/Month/Year so 01-01-2011 parses as 1 January (not 12 January).
   * Reason: A true Date type is required for building the model's Date table, for time-intelligence DAX (SAMEPERIODLASTYEAR, YoY growth).
   * Result: Both columns now store proper Date values spanning Jan 2011–Dec 2014.
2. Removed Postal Code column

   * Problem: Postal Code is null in 41,296 of 51,290 rows (\~81%). Most non-US markets never had postal codes captured, making the field unusable for geographic analysis.
   * Transformation: Removed the Postal Code column after confirming City/State/Country/Region already provide sufficient geographic detail.
   * Reason: A column that is over 80% blank adds noise and does not support any planned visual.
   * Result: Geographic analysis now relies only on the complete City/State/Country/Region/Market fields.
3. Clean whitespace in Product Name

   * Problem: 16 Product Name values contain leading or trailing spaces, which would make Power BI treat "Chairs " and "Chairs" as two distinct values.
   * Transformation: Applied Transform → Format → Trim to the Product Name column.
   * Reason: Prevents silent duplication(same conceptual category exists multiple times) in product-level visuals, category counts, and DISTINCTCOUNT measures.
   * Result: All product names normalized; distinct product counts are now accurate.
4. Investigate the Customer Name / Customer ID relationship

   * Problem: 795 distinct Customer Name values are each linked to more than one Customer ID, creating ambiguity for DISTINCTCOUNT measures.
   * Transformation: Grouped by Customer Name and counted distinct Customer IDs to confirm the pattern. We saw (check screenshot) that although there are 100% unique values in the 'Customer ID' column there are only has 524/1,000 unique values, then decided to treat Customer ID as the true unique key.
   * Reason: Determines the correct grain and key for DimCustomer and prevents undercounting customers by grouping on name.
   * Result: Customer ID confirmed as the reliable dimension key; Customer Name kept only as a display attribute.
5. Resolve the Region/Market hierarchy ambiguity

   * Problem: Region values are not unique across Market — "Central" appears under three different Markets (EU, US, LATAM), "North" under two (EU, LATAM), and "South" under three (US, LATAM, EU). Filtering or grouping by Region alone would silently merge three unrelated parts of the world into one "Central".
   * Transformation: Added a custom column, Market\_Region, concatenating Market and Region (\[Market] \& " - " \& \[Region]) to create a unique geographic key, keeping the original Region column for display only.
   * Reason: Prevents an ambiguous grouping where three genuinely different geographic areas would incorrectly appear as a single "Central" slice on maps and charts.
   * Result: A disambiguated geographic key ready for location-based grouping without misleading merges.
6. Split Order ID to extract the embedded region code, year, and sequence number

   * Problem: Order ID values (e.g. CA-2012-124891, IN-2013-77878) encode a region/country prefix, order year, and sequence number as one text string, none of which can be filtered or validated individually while stuck together.
   * Transformation: Select Order ID → Transform → Split Column → By Delimiter → -(Each Occurrence of the Delimeter).Compared the extracted OrderYear against the year from the actual Order Date as a data-integrity check.
   * Reason: Makes the embedded metadata usable on its own, and doubles as a sanity check confirming Order ID and Order Date were logged consistently.
   * Result: Three new fields available for filtering/validation, plus confirmation of whether Order ID and Order Date years agree.
7. Create a conditional "Profit Status" column

   * Problem: 13,212 of 51,290 rows (\~26%) are break-even or loss-making, but there is no field that lets a viewer immediately filter to losing transactions.
   * Transformation: Added a Conditional Column named Profit Status: if Profit > 0 then "Profitable" else if Profit = 0 then "Break-even" else "Loss-Making".
   * Reason: Turns a continuous number into a filterable category for the diagnostic dashboard page.
   * Result: A new three-value categorical column usable as a slicer, legend, or filter.
8. Create a custom "Shipping Duration (Days)" column

   * Problem: The dataset has both Order Date and Ship Date, but no direct measure of fulfillment speed.
   * Transformation: Added a Custom Column named ShippingDays calculated as Duration.Days(\[Ship Date] - \[Order Date]).
   * Reason: Enables analysis of fulfillment performance by Ship Mode, Region, or Order Priority.
   * Result: A new numeric column supporting logistics-focused KPIs and visuals.
