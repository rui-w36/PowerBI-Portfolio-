# Volkswagen Group (VWAGY) Financial Analysis Dashboard (2021–2025)


## 📊 Project Overview
This Power BI dashboard provides a deep-dive financial performance analysis of **Volkswagen Group** from 2021 to 2025. It tracks the group's transition through a critical period of electrification, highlighting the "Scissor Effect": **Stagnating revenue growth vs. heavy capital investment.**

### Key Findings:
* **Growth Stagnation:** Revenue growth nearly stalled by 2025 (+0.4%).
* **Margin Compression:** Net Margin halved from 2021 levels to 2.3% in 2025.
* **Cash Flow Stress:** Three consecutive years of negative Free Cash Flow (FCF) due to high CapEx (~8% of revenue), indicating a heavy reliance on external financing for EV/Software transformation.

---

## 🛠️ Methods (Power Query)
The project emphasizes **scalability** and **automation**. Data was sourced from Yahoo Finance with consistent line-item mapping across five years.

### Data Transformation Highlights:
* **Unpivoting:** Converted multi-dimensional "Wide" tables into a normalized "Long" format for efficient DAX calculations.
* **Dynamic Updates:** The ETL pipeline is designed for "One-Click Refresh." By simply adding a new data column to the source file, the model automatically integrates the new year's data without manual re-mapping.
* **Data Integrity:** Standardized data types and aligned account headers across Income Statement, Balance Sheet, and Cash Flow.

> **Note:** Balance Sheet data for FY2025 was not yet updated in the source at the time of report creation.

---

## 🖥️ Dashboard Structure
The report consists of four specialized analytical pages:

1. **Income Statement:** Analysis of revenue trends, COGS, and operational efficiency.
2. **Balance Sheet:** Visualizing asset structure, debt levels, and liquidity.
3. **Cash Flow:** Tracking the gap between Operating Cash Flow and heavy CapEx.
4. **DuPont Analysis:** A strategic breakdown of **Return on Equity (ROE)** to identify the true drivers of profitability and financial leverage.

---

## 🧪 Key DAX Measures
A robust library of DAX measures was developed for this report, including:
* **Profitability:** `Net Margin %`, `Gross Profit %`, `Operating Margin`.
* **Liquidity:** `Quick Ratio`, `Current Ratio`.
* **Growth:** `YoY Revenue Growth`, `YoY Net Income Change`.
* **Efficiency:** `Asset Turnover`, `Equity Multiplier`.

---

## 🚀 Interactive Dashboard
**Interactive Demo:** [https://app.powerbi.com/groups/me/reports/49c30ecb-bfc6-410d-ac26-03705a180104/46865ce90bba2bb8de88?experience=power-bi]

---

## 💡 Future Improvements

* Add 2025 Balance Sheet data once officially released by Yahoo Finance.
* Competitor benchmarking (Toyota vs. VW vs. Tesla).
