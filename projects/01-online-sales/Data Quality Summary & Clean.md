

The Online Retail dataset exhibits a well-structured transaction-level format that is highly representative of a real-world e-commerce system. Overall, the data quality is sufficient for in-depth business analysis, provided that several data quality considerations are appropriately handled.

---

# 1.Data Check

## 1.1 Data Loading & Structural Inspection
First, the dataset is loaded and its basic structure is verified to ensure row counts are reasonable (approx. 540k rows), field types are correct, and there are no unexpected object or float conversions.

```python
import pandas as pd
import numpy as np

# Load data
df = pd.read_excel("online_retail.xlsx")

# Basic structural checks
print(df.head())
print(df.shape)
print(df.info())
```

## 1.2 Missing Value Analysis
The dataset contains missing values in the `CustomerID` field, accounting for approximately 25% of all records. This pattern is consistent with typical e-commerce scenarios where transactions may occur without registered customer identification (e.g., guest checkouts). As a result, records without `CustomerID` are excluded from customer-level analyses but retained for aggregate sales analysis where applicable. A small number of records also contain missing values in the `Description` field.

```python
# Check for missing values
missing_values = df.isnull().sum().sort_values(ascending=False)
print(missing_values)

# Business Logic: 
# Missing CustomerID likely represents guest users or unrecorded transactions.
# Decision: Do not delete immediately; explain first, then decide on exclusion based on analysis type.
```

## 1.3 Product Identifier Consistency (StockCode vs. Description)
A consistency check revealed that a single `StockCode` may correspond to multiple product descriptions, likely due to variations in manual data entry rather than actual product differences. Consequently, `StockCode` is treated as the primary product identifier, and missing product descriptions are not imputed and are excluded from descriptive text-based analysis.

```python
# Verify if StockCode maps uniquely to Description
desc_per_stock = (
    df.groupby("StockCode")["Description"]
      .nunique()
      .reset_index(name="description_count")
)

# View top inconsistencies
print(desc_per_stock.sort_values("description_count", ascending=False).head(10))

# Identify specific anomalous products
anomalous_products = desc_per_stock[desc_per_stock["description_count"] > 1]
print(anomalous_products)
```

## 1.4 Duplicate Record Handling
Duplicate records were identified, representing less than 1% of the dataset. These duplicates are considered technical artifacts of data extraction and are removed during the data cleaning stage.

```python
# Check for duplicate rows
duplicate_count = df.duplicated().sum()
print(f"Number of duplicate records: {duplicate_count}")

# Inspect duplicates if necessary
if duplicate_count > 0:
    print(df[df.duplicated()].head())
    # Decision: Remove duplicates, keeping the first occurrence
    df = df.drop_duplicates(keep='first')
```

## 1.5 Anomaly Detection: Quantity and Unit Price
The dataset includes transactions with negative quantities and a subset of records with zero or negative unit prices. 

*   **Negative Quantities:** Cross-validation confirms a strong relationship between negative quantities and invoice numbers starting with "C". These are preserved and flagged as cancellations.
*   **Non-positive Unit Prices:** These observations lack clear business meaning (likely system test records or errors) and are removed.

```python
# Inspect Quantity anomalies
print(df["Quantity"].describe())
print(df[df["Quantity"] <= 0].head())

# Inspect UnitPrice anomalies (System errors/Test data)
invalid_price_records = df[df["UnitPrice"] <= 0]
print(invalid_price_records.head())

# Action: Remove records with UnitPrice <= 0 as they lack business meaning
df = df[df["UnitPrice"] > 0]
```

## 1.6 Cancellation Identification & Validation
Instead of treating negative quantities as errors, this project explicitly identifies them as order cancellations or returns. We verify the hypothesis that `Quantity < 0` is equivalent to an Invoice Number starting with "C".

```python
# Feature Engineering: Identify cancelled orders based on InvoiceNo prefix
df["IsCancelled"] = df["InvoiceNo"].astype(str).str.startswith("C")

# Validate the distribution
print(df["IsCancelled"].value_counts())

# Step 1: Cross-tabulation to verify the relationship between Negative Quantity and Cancellation Flag
crosstab = pd.crosstab(df["Quantity"] < 0, df["IsCancelled"])
print(crosstab)

# Step 2: Check for counter-examples (Critical Validation)
# Case A: Negative quantity but NOT marked as cancelled (Invoice does not start with 'C')
false_negatives = df[(df["Quantity"] < 0) & (~df["IsCancelled"])]
print("Negative Qty but not Cancelled Flag:")
print(false_negatives.head())

# Case B: Marked as cancelled (starts with 'C') but Quantity is non-negative
false_positives = df[(df["Quantity"] >= 0) & (df["IsCancelled"])]
print("Cancelled Flag but Non-Negative Qty:")
print(false_positives.head())

# Conclusion: The strong pattern supports interpreting negative quantities as cancellations.
# These records are preserved and flagged, not deleted.
```

## 1.7 Temporal and Geographical Consistency
Finally, time fields are checked for reasonableness within the 2010–2011 range, and country fields are inspected for spelling anomalies or extreme concentration.

```python
# Convert and check date range
df["InvoiceDate"] = pd.to_datetime(df["InvoiceDate"])
print(f"Date Range: {df['InvoiceDate'].min()} to {df['InvoiceDate'].max()}")

# Check Country consistency
print(f"Unique Countries: {df['Country'].nunique()}")
print(df["Country"].value_counts().head(10))
```

## Conclusion
In summary, instead of indiscriminately removing all anomalous values, this project distinguishes between true data errors and business-relevant events. By explicitly identifying and preserving meaningful anomalies such as order cancellations through the code steps above, the cleaned dataset maintains both analytical integrity and business interpretability.

---

# 2. Data Analysis

## 2.1 Data Cclean
Following the initial quality inspection, we proceed to the formal data cleaning stage. The objective is to transform the raw dataset into a reliable analytical base (`df_clean`) by executing the decisions made in the previous section. This involves removing technical duplicates and invalid pricing records, while explicitly flagging cancellation events rather than deleting them, ensuring the cleaned dataset reflects true business operations.

```python
# Step 1: Remove completely duplicate records identified earlier
# These are considered extraction artifacts and do not represent unique transactions.
df_clean = df.drop_duplicates()

# Step 2: Remove Invalid Price Records
# Records with UnitPrice <= 0 are removed as they lack business meaning (likely test data or errors).
# Note: This step was identified in the Anomaly Detection phase.
df_clean = df_clean[df_clean["UnitPrice"] > 0]

# Step 3: Create Cancellation Flag
# Instead of removing negative quantity records, we explicitly label them.
# Invoice numbers starting with 'C' indicate cancellations/returns.
# This flag allows us to filter them out for sales analysis while retaining them for return rate analysis.
df_clean["IsCancelled"] = df_clean["InvoiceNo"].astype(str).str.startswith("C")


```

## 2.2 Data Analysis
With the data cleaned, we focus on calculating key performance indicators (KPIs) for active transactions. To ensure accuracy, all cancellation records (`IsCancelled == True`) are excluded from this specific analysis. We derive a `Revenue` column to quantify sales value and compute aggregate metrics including Total Revenue, Total Order Count, and Average Order Value (AOV). Additionally, we aggregate sales by month to identify temporal trends.

```python
# Filter out cancelled orders for core sales analysis
df_analysis = df_clean[~df_clean["IsCancelled"]].copy()

# Feature Engineering: Calculate Revenue per transaction line
df_analysis["Revenue"] = df_analysis["Quantity"] * df_analysis["UnitPrice"]

# Calculate Core KPIs
total_revenue = df_analysis["Revenue"].sum()
total_orders = df_analysis["InvoiceNo"].nunique()
aov = total_revenue / total_orders if total_orders > 0 else 0
total_quantity = df_analysis["Quantity"].sum()

print(f"Total Revenue: ${total_revenue:,.2f}")
print(f"Total Orders: {total_orders}")
print(f"Average Order Value (AOV): ${aov:,.2f}")
print(f"Total Quantity Sold: {total_quantity}")

# Time Series Analysis: Monthly Sales Trend
# Convert InvoiceDate to Period for easier monthly grouping
df_analysis["YearMonth"] = df_analysis["InvoiceDate"].dt.to_period("M")

monthly_sales = (
    df_analysis
    .groupby("YearMonth")["Revenue"]
    .sum()
    .reset_index()
    .sort_values("YearMonth")
)


```

## 2.3 RFM Analysis
To gain deeper insights into customer behavior, we perform an RFM (Recency, Frequency, Monetary) analysis. This method segments customers based on:
*   **Recency (R):** How recently a customer made a purchase.
*   **Frequency (F):** How often they purchase.
*   **Monetary (M):** How much they spend.

We first aggregate transaction data at the `CustomerID` level, excluding cancellations. A robust scoring system (1-5 scale) is applied to each metric using percentiles. Finally, custom business rules map these scores to meaningful segments such as "Champions," "Loyal Customers," and "At Risk," enabling targeted marketing strategies.

```python
# Prepare data for RFM: Exclude cancellations and ensure CustomerID is present
# Note: We use df_clean to ensure we have the latest state, then filter non-cancelled
rfm_data = df_clean[~df_clean["IsCancelled"]].copy()
rfm_data = rfm_data.dropna(subset=['CustomerID']) # Ensure no missing IDs for customer analysis

# Define Reference Date (latest transaction date in the dataset)
reference_date = rfm_data['InvoiceDate'].max()

# Aggregate metrics per Customer
# Recency: Days since last purchase
# Frequency: Number of unique invoices (orders)
# Monetary: Total revenue generated
# Note: We calculate Revenue on the fly for aggregation to ensure consistency
rfm_data['Revenue'] = rfm_data['Quantity'] * rfm_data['UnitPrice']

rfm = rfm_data.groupby('CustomerID').agg({
    'InvoiceDate': lambda x: (reference_date - x.max()).days, # Recency
    'InvoiceNo': 'nunique',                                   # Frequency
    'Revenue': 'sum'                                          # Monetary
}).reset_index()

rfm.columns = ['CustomerID', 'Recency', 'Frequency', 'Monetary']

# Robust Scoring Function
# Assigns scores 1-5 based on percentiles. 
# For Recency, lower is better (ascending=True maps low values to high scores).
# For Frequency and Monetary, higher is better (ascending=False maps high values to high scores).
def rfm_score(series, ascending=True):
    percentiles = series.rank(pct=True, method='min')
    if ascending:
        # Lower value -> Higher score (e.g., recent buyers get 5)
        return pd.cut(percentiles, bins=[0, 0.2, 0.4, 0.6, 0.8, 1.0], labels=[5, 4, 3, 2, 1], include_lowest=True).astype(int)
    else:
        # Higher value -> Higher score
        return pd.cut(percentiles, bins=[0, 0.2, 0.4, 0.6, 0.8, 1.0], labels=[1, 2, 3, 4, 5], include_lowest=True).astype(int)

# Apply Scoring
rfm['R'] = rfm_score(rfm['Recency'], ascending=True)
rfm['F'] = rfm_score(rfm['Frequency'], ascending=False)
rfm['M'] = rfm_score(rfm['Monetary'], ascending=False)

# Customer Segmentation Logic
# Define business rules to map R, F, M scores to specific segments
def get_segment(row):
    r, f, m = row['R'], row['F'], row['M']
    if r >= 4 and f >= 4 and m >= 4:
        return 'Champions'
    elif r >= 2 and f >= 3 and m >= 3:
        return 'Loyal Customers'
    elif r >= 3 and f <= 3 and m <= 3:
        return 'Potential Loyalists'
    elif r >= 4 and f == 1 and m <= 2:
        return 'Recent Customers'
    elif r == 3 and f == 1 and m == 1:
        return 'Promising'
    elif 2 <= r <= 3 and 2 <= f <= 3 and 2 <= m <= 3:
        return 'Customers Needing Attention'
    elif r <= 2 and f >= 2 and m >= 2:
        return 'At Risk'
    elif r == 1 and f >= 4 and m >= 4:
        return "Can't Lose Them"
    elif r <= 2 and f <= 2 and m <= 2:
        return 'Hibernating'
    elif r == 1 and f == 1 and m == 1:
        return 'Lost'
    else:
        return 'Others'

# Apply segmentation
rfm['Segment'] = rfm.apply(get_segment, axis=1)


```

## Conclusion
By systematically executing data cleaning and advanced analysis, we have transformed the raw Online Retail dataset into a strategic asset. The cleaning process ensured data integrity by distinguishing between errors and valid business events (cancellations). Subsequent analysis provided actionable insights:
1.  **Operational Clarity:** Clear KPIs (Revenue, AOV) and monthly trends establish a baseline for performance tracking.
2.  **Customer Intelligence:** The RFM segmentation identifies high-value "Champions" and at-risk customers, enabling precise, data-driven marketing interventions.

This end-to-end workflow demonstrates how rigorous data preparation directly enhances the quality and interpretability of business analytics.
