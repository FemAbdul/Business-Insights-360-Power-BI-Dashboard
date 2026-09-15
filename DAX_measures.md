# DAX Measures Reference — Business Insights 360

This documents the core measure logic behind the report's P&L, margin, and forecast-accuracy calculations. Replace the placeholder table/column names below with the actual names from your model before publishing — this file is meant as documentation, not a 1:1 export of your `.pbix`.

## P&L Bridge

```dax
Gross Sales = SUM(Sales[GrossSalesAmount])

Pre Invoice Deduction = SUM(Sales[PreInvoiceDeductionAmount])

Net Invoice Sales = [Gross Sales] - [Pre Invoice Deduction]

Post Discounts = SUM(Sales[DiscountAmount])

Post Deductions = SUM(Sales[OtherDeductionAmount])

Total Post Invoice Deduction = [Post Discounts] + [Post Deductions]

Net Sales = [Net Invoice Sales] - [Total Post Invoice Deduction]

Manufacturing Cost = SUM(Sales[ManufacturingCost])
Freight Cost        = SUM(Sales[FreightCost])
Other Cost          = SUM(Sales[OtherCost])

Total COGS = [Manufacturing Cost] + [Freight Cost] + [Other Cost]

Gross Margin = [Net Sales] - [Total COGS]

Gross Margin % = DIVIDE([Gross Margin], [Net Sales])

GM per Unit = DIVIDE([Gross Margin], SUM(Sales[UnitsSold]))

Operational Expense = SUM(Sales[OpEx])

Net Profit = [Gross Margin] - [Operational Expense]

Net Profit % = DIVIDE([Net Profit], [Net Sales])
```

## Variance vs Benchmark / Last Year / Target

```dax
Net Sales BM = CALCULATE([Net Sales], 'Benchmark Table')

Net Sales Chg = [Net Sales] - [Net Sales BM]

Net Sales Chg % = DIVIDE([Net Sales Chg], [Net Sales BM])

Net Sales LY =
CALCULATE(
    [Net Sales],
    SAMEPERIODLASTYEAR('Date'[Date])
)

Net Sales vs LY % = DIVIDE([Net Sales] - [Net Sales LY], [Net Sales LY])

-- "vs LY" / "vs Target" toggle pattern (field parameter or disconnected slicer table)
Comparison Value =
SWITCH(
    SELECTEDVALUE('Comparison Toggle'[Selection]),
    "vs LY", [Net Sales LY],
    "vs Target", [Net Sales Target],
    [Net Sales BM]
)
```

## Forecast Accuracy & Risk Flagging

```dax
Forecast Accuracy =
1 - DIVIDE(
        SUMX(Forecast, ABS(Forecast[ForecastQty] - Forecast[ActualQty])),
        SUM(Forecast[ActualQty])
    )

Net Error = SUMX(Forecast, Forecast[ForecastQty] - Forecast[ActualQty])

Net Error % = DIVIDE([Net Error], SUM(Forecast[ActualQty]))

ABS Error = SUMX(Forecast, ABS(Forecast[ForecastQty] - Forecast[ActualQty]))

Risk Flag =
SWITCH(
    TRUE(),
    [Net Error] > 0, "EI",   -- Excess Inventory: over-forecasted
    [Net Error] < 0, "OOS",  -- Out of Stock: under-forecasted
    "On Target"
)
```

## Revenue Contribution & Market Share

```dax
RC % =
DIVIDE(
    [Net Sales],
    CALCULATE([Net Sales], ALL(Customer))   -- or ALL(Product) depending on context
)

Atliq Market Share % =
DIVIDE(
    CALCULATE([Net Sales], Product[Brand] = "Atliq"),
    CALCULATE([Net Sales], ALL(Product[Brand]))
)
```

## Notes

- Measures assume a star schema: a central `Sales` fact table plus `Date`, `Customer`, `Product` (with `Segment`/`Category` hierarchy), `Region/Market`, and a separate `Forecast` fact table joined on Date/Product/Customer.
- The **vs LY / vs Target / vs Benchmark** toggle on each page is typically built with a disconnected "Comparison Toggle" table and a `SWITCH` measure, so every KPI card and chart responds to the same selector.
- `EI` (Excess Inventory) and `OOS` (Out of Stock) thresholds can be tuned — consider adding a **Target Gap Tolerance** parameter (as used on the Sales/Marketing performance-matrix pages) to control how far off-target a point must be before it's flagged.
