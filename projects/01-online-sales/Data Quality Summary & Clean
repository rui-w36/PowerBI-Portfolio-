

The Online Retail dataset exhibits a well-structured transaction-level format that is highly representative of a real-world e-commerce system. Overall, the data quality is sufficient for in-depth business analysis, provided that several data quality considerations are appropriately handled.

## 1. Data Loading & Structural Inspection
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

## 2. Missing Value Analysis
The dataset contains missing values in the `CustomerID` field, accounting for approximately 25% of all records. This pattern is consistent with typical e-commerce scenarios where transactions may occur without registered customer identification (e.g., guest checkouts). As a result, records without `CustomerID` are excluded from customer-level analyses but retained for aggregate sales analysis where applicable. A small number of records also contain missing values in the `Description` field.

```python
# Check for missing values
missing_values = df.isnull().sum().sort_values(ascending=False)
print(missing_values)

# Business Logic: 
# Missing CustomerID likely represents guest users or unrecorded transactions.
# Decision: Do not delete immediately; explain first, then decide on exclusion based on analysis type.
```

## 3. Product Identifier Consistency (StockCode vs. Description)
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

## 4. Duplicate Record Handling
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

## 5. Anomaly Detection: Quantity and Unit Price
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

## 6. Cancellation Identification & Validation
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

## 7. Temporal and Geographical Consistency
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
