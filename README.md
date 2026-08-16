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
### Data Modeling

FactOrders contains transactional information — one row per order line item — and forms the centre of the model. DimCustomer, DimProduct, DimLocation, and DimDate provide descriptive attributes used to filter and group the fact table. One-to-many relationships were established between each dimension and the fact table, with a single cross-filter direction flowing from each dimension into the fact table.



Why FactOrders was selected as the fact table: it is the most granular table available (51,280 order-line rows) and holds every core numeric measure the analysis depends on ; Sales, Quantity, Discount, Profit, and Shipping Cost.



Why each dimension was created:



* DimCustomer — needed to analyze profitability and behaviour by customer segment, using the verified Customer ID as the key (not Customer Name, after the mismatch investigated during Power Query).
* DimProduct — needed for category and sub-category level profitability analysis, one of the project's core analytical questions.
* DimLocation — needed given the dataset's global scope (147 countries, 7 markets), and specifically to resolve the Region/Market ambiguity discovered during cleaning.
* DimDate — required for correct time-intelligence DAX (Previous Year Sales, YoY Growth%) and for consistent Year/Quarter/Month slicers not limited only to dates with a transaction.



Relationships, cardinality, and filter direction: all four dimensions relate to FactOrders one-to-many, with a single cross-filter direction from dimension to fact — the standard star-schema pattern, keeping every filter path unambiguous without needing any bidirectional relationships.



Modelling challenges encountered:



* Region/Market ambiguity — Region values are not globally unique. Resolved by building DimLocation's key from the full Market+Region+Country+State+City combination rather than Region alone.
* Customer Name vs Customer ID — 795 customer names mapped to more than one Customer ID. Resolved by using CustomerID, not CustomerName, as the dimension key.
* Product ID is not reliably unique — 457 Product IDs (\~4.4% of all products) are each attached to two different products. Resolved the same way as the Region/Market issue: built a composite ProductKey (Product ID + Product Name) instead of trusting Product ID alone, avoiding an unintended many-to-many relationship.
* Ship Date — the fact table has two dates (Order Date, Ship Date), but only Order Date is related to DimDate. Formally relating both would require marking one relationship inactive and invoking USERELATIONSHIP() in every Ship-Date-based measure ,added complexity not justified by this project's core questions, so Ship Date stays a plain fact attribute, used only to calculate ShippingDays.

### DAX Measures

1. Profit Margin %

   * Calculates: Total Profit as a percentage of Total Sales for the current filter context.
   * Why useful: Sales alone can hide low-margin or loss-making areas. This measure exposes true profitability.
   * Main DAX functions: DIVIDE (with 0 fallback to avoid divide-by-zero errors).
   * Filter context: Fully responsive — both \[Total Profit] and \[Total Sales] re-aggregate under the active filters before the division.
   * Used in: Executive Overview (headline KPI card) and Diagnostic page (by Category/Market/Segment).
2. Lost Sales Value

   * Calculates: Total Sales from transactions flagged as Loss-Making (Profit ≤ 0).
   * Why useful: Converts the abstract “26% of transactions lose money” into a concrete dollar amount management can act on.
   * Main DAX functions: CALCULATE modifying \[Total Sales] with a filter on the Profit Status column.
   * Filter context: CALCULATE adds the Profit Status filter on top of any existing context (e.g. Region or Date).
   * Used in: Diagnostic page as a KPI card next to Total Sales.
3. High-Discount Profit Margin %

   * Calculates: Profit Margin % calculated only on transactions with Discount above 30%.
   * Why useful: Directly tests whether heavy discounting drives the loss-making transactions. The gap versus overall margin is the key insight.
   * Main DAX functions: VAR (for the threshold), CALCULATE, FILTER (row-level numeric condition).
   * Filter context: FILTER iterates under the current context, then CALCULATE recomputes the margin only on the high-discount rows within that context.
   * Used in: Diagnostic page, side-by-side with the standard Profit Margin % card.
4. Category Profit Rank

   * Calculates: Rank of the current Category by Total Profit (1 = most profitable), ignoring any Category filter on the visual.
   * Why useful: A raw profit number does not show relative standing. Rank instantly shows which categories lead or lag.
   * Main DAX functions: RANKX, ALL (to rank across all Categories), CALCULATE (for context transition).
   * Filter context: ALL(DimProduct\[Category]) deliberately overrides Category filters so ranking always runs across all categories. Other filters (Region, Date) still apply.
   * Used in: Product Analysis page, in a table alongside Category Sales Rank.
5. YoY Sales Growth %

   * Calculates: Percentage change in Sales versus the same period one year earlier.
   * Why useful: Answers whether the business is actually growing, not just the absolute Sales figure for a single year.
   * Main DAX functions: SAMEPERIODLASTYEAR, CALCULATE, DIVIDE.
   * Filter context: Requires DimDate to be marked as the official Date table with a continuous calendar. SAMEPERIODLASTYEAR shifts the current date context back one year.
   * Used in: Executive Overview as the headline trend indicator (line chart + KPI card).
6. Performance Tier

   * Calculates: A text label (“Loss-Making” / “Low Margin” / “Healthy” / “Strong”) based on the current Profit Margin %.
   * Why useful: Gives managers an instant plain-language read on a KPI card without interpreting a raw percentage.
   * Main DAX functions: SWITCH combined with TRUE() (condition-based pattern) that internally evaluates \[Profit Margin %].
   * Filter context: Inherits whatever context \[Profit Margin %] is evaluated under, so the label updates correctly for the overall business or any sliced view.
   * Used in: Executive Overview (colour-coded KPI card) and potentially for conditional formatting on Diagnostic page tables.

