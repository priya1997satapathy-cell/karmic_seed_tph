# Fulfillment Hub — Power BI Fulfillment Operations Dashboard

## Overview

This project is a Power BI solution designed for **XYZ, an e-commerce fulfillment business**, to improve operational visibility across order fulfillment, SLA performance, exceptions, and inventory-related issues.

XYZ processes e-commerce orders through:

**Order Received → Processing → Picking → Packing → Staging → Pickup → Delivery**

The business challenges include limited order visibility, priority orders being mixed with regular orders, SLA delays, inventory discrepancies, warehouse transfers, misplaced packages, missed pickups, wrong product/variant shipments, and operational exceptions being handled informally.

The objective is to provide a **simple, practical Power BI application** that helps operational teams monitor fulfillment, identify exceptions, analyze performance, investigate individual orders, and understand inventory-related problems.

---

## Report Structure

| Page | Name | Purpose |
|---|---|---|
| 1 | **Fulfillment Control Centre** | Monitor orders and issues requiring attention |
| 2 | **Fulfillment Performance** | Analyze historical fulfillment performance |
| 3 | **Order at a Glance** | Investigate an individual order |
| 4 | **Inventory & Warehouse** | Analyze inventory-related fulfillment problems |

---

# Page 1 — Fulfillment Control Centre

## Purpose

The main operational monitoring page.

**Business question:** What needs attention?

### Main visuals

- Total Orders KPI
- Priority Orders KPI
- Open Orders KPI
- At Risk Orders KPI
- SLA Breached KPI
- Open Exceptions KPI
- Fulfillment Pipeline
- Priority Orders Requiring Attention table
- Orders by Exception
- Inventory alerts

## Core Measures

### Total Orders

```DAX
Total Orders =
COUNTROWS(FactOrder)
```

Counts total orders in the current filter context.

### Priority Orders

```DAX
Priority Orders =
CALCULATE(
    [Total Orders],
    FactOrder[IsPriority] = TRUE()
)
```

Counts priority orders.

### Open Orders

```DAX
Open Orders =
CALCULATE(
    [Total Orders],
    FactOrder[OrderStatus] <> "Delivered",
    FactOrder[OrderStatus] <> "Cancelled"
)
```

Counts orders that have not been delivered or cancelled.

### At Risk Orders

Historical data is used instead of `NOW()` because the dataset covers January–June 2026.

An order is At Risk when it is not cancelled, has not reached staging, processing has started, picking has completed, and the SLA deadline falls between processing start and picking end.

```DAX
At Risk Orders =
CALCULATE(
    [Total Orders],
    FILTER(
        FactOrder,
        FactOrder[OrderStatus] <> "Cancelled"
            &&
        ISBLANK(FactOrder[StagingStartTime])
            &&
        NOT ISBLANK(FactOrder[ProcessingStartTime])
            &&
        NOT ISBLANK(FactOrder[PickingEndTime])
            &&
        FactOrder[SLADeadline] >= FactOrder[ProcessingStartTime]
            &&
        FactOrder[SLADeadline] <= FactOrder[PickingEndTime]
    )
)
```

### SLA Breached

An order is breached when it reaches staging after its SLA deadline.

```DAX
SLA Breached =
CALCULATE(
    [Total Orders],
    FILTER(
        FactOrder,
        FactOrder[OrderStatus] <> "Cancelled"
            &&
        NOT ISBLANK(FactOrder[StagingStartTime])
            &&
        FactOrder[StagingStartTime] > FactOrder[SLADeadline]
    )
)
```

### Open Exceptions

```DAX
Open Exceptions =
CALCULATE(
    [Total Orders],
    FactOrder[HasException] = TRUE(),
    FactOrder[ExceptionStatus] = "Open"
)
```

Counts currently open exception orders.

## Fulfillment Pipeline

The pipeline measures how many orders have progressed through each fulfillment milestone.

```DAX
Received Pipeline =
[Total Orders]
```

```DAX
Processing Pipeline =
CALCULATE(
    [Total Orders],
    NOT ISBLANK(FactOrder[ProcessingStartTime])
)
```

```DAX
Picking Pipeline =
CALCULATE(
    [Total Orders],
    NOT ISBLANK(FactOrder[PickingStartTime])
)
```

```DAX
Packing Pipeline =
CALCULATE(
    [Total Orders],
    NOT ISBLANK(FactOrder[PackingStartTime])
)
```

```DAX
Staging Pipeline =
CALCULATE(
    [Total Orders],
    NOT ISBLANK(FactOrder[StagingStartTime])
)
```

Stage table:

```DAX
FulfillmentStages =
DATATABLE(
    "Stage", STRING,
    "SortOrder", INTEGER,
    {
        {"Received", 1},
        {"Processing", 2},
        {"Picking", 3},
        {"Packing", 4},
        {"Staging", 5}
    }
)
```

Pipeline mapping:

```DAX
Pipeline Orders =
SWITCH(
    SELECTEDVALUE(FulfillmentStages[Stage]),
    "Received", [Received Pipeline],
    "Processing", [Processing Pipeline],
    "Picking", [Picking Pipeline],
    "Packing", [Packing Pipeline],
    "Staging", [Staging Pipeline]
)
```

---

# Page 2 — Fulfillment Performance

## Purpose

Historical performance analysis.

**Business question:** How is fulfillment performing over time?

### Main visuals

- Orders Fulfilled Over Time
- SLA Performance by Month
- Average Fulfillment Time by Stage
- Exceptions Over Time
- Inventory Issues by Type

## SLA Measures

### Completed On Time

An order reaches staging on or before its SLA deadline.

```DAX
Completed On Time =
CALCULATE(
    [Total Orders],
    FILTER(
        FactOrder,
        FactOrder[OrderStatus] <> "Cancelled"
            &&
        NOT ISBLANK(FactOrder[StagingStartTime])
            &&
        FactOrder[StagingStartTime] <= FactOrder[SLADeadline]
    )
)
```

### Completed Late

An order reaches staging after its SLA deadline.

```DAX
Completed Late =
CALCULATE(
    [Total Orders],
    FILTER(
        FactOrder,
        FactOrder[OrderStatus] <> "Cancelled"
            &&
        NOT ISBLANK(FactOrder[StagingStartTime])
            &&
        FactOrder[StagingStartTime] > FactOrder[SLADeadline]
    )
)
```

### SLA Compliance %

```DAX
SLA Compliance % =
DIVIDE(
    [Completed On Time],
    [Completed On Time] + [Completed Late]
)
```

### Example stage-duration measure

```DAX
Avg Processing Time =
AVERAGEX(
    FILTER(
        FactOrder,
        NOT ISBLANK(FactOrder[ProcessingStartTime])
            &&
        NOT ISBLANK(FactOrder[ProcessingEndTime])
    ),
    DATEDIFF(
        FactOrder[ProcessingStartTime],
        FactOrder[ProcessingEndTime],
        MINUTE
    )
)
```

Equivalent calculations can be created for picking and packing.

---

# Date Model

The report uses a dedicated Date dimension.

```text
DimDate[Date]  1 ───────── *  FactOrder[OrderDate]
```

Relationship settings:

- Cardinality: One-to-many
- Cross-filter direction: Single
- Active relationship: Yes

`MonthName` is sorted by `MonthNumber` so months appear chronologically.

Dataset period:

**January 2026 – June 2026**

---

# Page 3 — Order at a Glance

## Purpose

A drill-through page for investigating a single order.

**Business question:** What happened to this specific order?

### Drill-through field

```text
FactOrder[OrderID]
```

The page is intended to contain:

### Order Header

- Order ID
- Priority
- Order Status
- Order Date

### Customer & Product

- Customer
- City
- Customer Segment
- Product
- Category
- Brand
- Variant

### Fulfillment Timeline

- Order Received
- Processing
- Picking
- Packing
- Staging
- Pickup
- Delivery

Missing timestamps are intentionally retained because they can represent genuine operational exceptions.

### SLA Information

- SLA Deadline
- SLA Status
- Staging Time

Example:

```DAX
SLA Status =
VAR StagingTime = SELECTEDVALUE(FactOrder[StagingStartTime])
VAR SLADeadline = SELECTEDVALUE(FactOrder[SLADeadline])
VAR OrderStatus = SELECTEDVALUE(FactOrder[OrderStatus])
RETURN
SWITCH(
    TRUE(),
    OrderStatus = "Cancelled", "Cancelled",
    ISBLANK(StagingTime), "Not Yet Staged",
    StagingTime <= SLADeadline, "Completed On Time",
    StagingTime > SLADeadline, "Completed Late"
)
```

### Exception Details

- Has Exception
- Exception Type
- Exception Status
- Inventory Issue
- Inventory Issue Type
- Transfer Required

---

# Page 4 — Inventory & Warehouse

## Purpose

Analyze how inventory availability affects fulfillment.

**Business question:** How is inventory affecting fulfillment?

The page focuses on:

- Inventory issues
- Warehouse transfers
- Inventory shortages
- Inventory discrepancies
- Product categories affected
- Products causing inventory problems

## Planned KPI Measures

### Inventory Issue Orders

```DAX
Inventory Issue Orders =
CALCULATE(
    [Total Orders],
    FactOrder[InventoryIssue] = TRUE()
)
```

### Transfer Required Orders

```DAX
Transfer Required Orders =
CALCULATE(
    [Total Orders],
    FactOrder[TransferRequired] = TRUE()
)
```

### Inventory Shortage Orders

```DAX
Inventory Shortage Orders =
CALCULATE(
    [Total Orders],
    FactOrder[InventoryIssueType] = "Inventory Shortage"
)
```

### Inventory Discrepancy Orders

```DAX
Inventory Discrepancy Orders =
CALCULATE(
    [Total Orders],
    FactOrder[InventoryIssueType] = "Inventory Discrepancy"
)
```

### Planned visuals

- Inventory Issues by Type
- Transfer Required by Month
- Inventory Issues by Product Category
- Products With Inventory Issues

---

# Data Model

The project uses a simple **Star Schema**.

```text
                    DimCustomer
                         │
                         ▼
DimProduct ───────── FactOrder ───────── DimWarehouse
                         │
                         ▼
                      DimDate
```

## FactOrder

One row represents **one customer order**.

The project intentionally uses:

> **1 Order = 1 Product**

There is no separate Order Lines table.

## DimCustomer

Contains customer attributes such as:

- Customer ID
- Customer Name
- City
- State
- Pincode
- Customer Segment

## DimProduct

Contains:

- Product ID
- Product Name
- Category
- Subcategory
- Brand
- Variant
- Unit Price
- Reorder Level
- Active Status

## DimWarehouse

Contains:

- Main Warehouse
- Secondary Warehouse

Orders are fulfilled and shipped from the main warehouse. The secondary warehouse is used for storage and stock transfers.

## DimDate

Contains calendar attributes for time-based analysis.

---

# Dataset

The data is synthetic and was created specifically for this project.

| Dataset | Size |
|---|---:|
| FactOrder | 84,000 orders |
| DimCustomer | 20,000 customers |
| DimProduct | 750 products |
| DimWarehouse | 2 warehouses |
| DimDate | 181 dates |
| Period | Jan–Jun 2026 |

---

# Synthetic Operational Scenarios

The dataset intentionally contains realistic operational variation:

- Priority orders
- Inventory shortages
- Inventory discrepancies
- Warehouse transfer delays
- Processing delays
- Picking delays
- Packing delays
- Package misplaced
- Missed pickup
- Wrong product / variant
- Damaged package
- SLA delays
- Cancellations

Missing downstream timestamps can therefore represent a legitimate operational state rather than simply missing data.

---

# Data Quality Rules

Operational timestamps follow logical dependencies:

```text
Processing Start >= Order Received
Picking Start >= Processing End
Picking End >= Picking Start
Packing Start >= Picking End
Packing End >= Packing Start
Staging Start >= Packing End
Pickup >= Staging Start
Delivery >= Pickup
```

If an exception prevents an order from progressing, later timestamps may remain blank.

---

# Historical SLA Approach

The dataset is historical, covering January–June 2026.

The report does **not** use `NOW()` for historical SLA evaluation because comparing old SLA deadlines with the current date would classify historical orders as overdue simply because the current date is later.

Instead, SLA performance is based on the actual operational milestone:

```text
StagingStartTime <= SLADeadline
```

means the fulfillment SLA was achieved on time.

---

# Key Design Decisions

## No Courier Analysis

Courier analysis was intentionally excluded. The project focuses on fulfillment operations, SLA performance, exceptions, inventory, and warehouse movement.

## No Separate Inventory Fact

A separate inventory fact table was intentionally not created. Inventory-related order attributes are used to analyze how inventory problems affect fulfillment.

This keeps the solution simple and appropriate for the take-home assignment.

## No Order Lines

Each order contains one product. This avoids unnecessary complexity while still supporting product and inventory analysis.

---

# Business Value

The dashboard provides XYZ with a simple operational BI application that can:

- Identify priority orders requiring attention
- Surface SLA risks and breaches
- Monitor the fulfillment pipeline
- Analyze historical fulfillment performance
- Identify recurring operational exceptions
- Investigate individual orders
- Identify inventory-related fulfillment problems
- Highlight orders requiring warehouse transfer
- Identify product categories contributing to inventory issues

The solution is focused on **visibility, exception management, and operational decision support**.

---

# Limitations

This is a synthetic take-home project rather than a production fulfillment management system.

Limitations include:

- Inventory is analyzed through order-level attributes rather than a real-time inventory ledger.
- The dataset is synthetic.
- The model does not include detailed order-line level fulfillment.
- Courier performance is outside the scope.
- SLA definitions are simplified for demonstration.
- The report provides operational visibility rather than automated workflow execution.

---

# Future Enhancements

With access to real operational systems, the solution could be extended with:

- Real-time inventory integration
- Order-line level tracking
- Barcode/scanning events
- Courier pickup monitoring
- Warehouse task assignment
- Automated exception alerts
- Real-time SLA countdowns
- Inventory transaction history
- Stock movement tracking
- Root-cause analysis using operational event logs

These enhancements are outside the scope of this take-home project.

---

# Tools & Technologies

- **Power BI**
- **DAX**
- **Power Query**
- **Star Schema / Dimensional Modeling**
- Synthetic data generation

---

# Recommended Repository Structure

```text
Fulfillment-Hub/
│
├── README.md
│
├── PowerBI/
│   └── Fulfillment_Hub.pbix
│
├── Data/
│   ├── FactOrder.csv
│   ├── DimCustomer.csv
│   ├── DimProduct.csv
│   ├── DimWarehouse.csv
│   ├── DimDate.csv
│   └── DataDictionary.csv
│
└── Documentation/
    └── DataDictionary.csv
```

---

# Project Outcome

The final application follows four stages of operational analysis:

**Monitor → Analyze → Investigate → Diagnose**

```text
Fulfillment Control Centre
            ↓
Fulfillment Performance
            ↓
Order at a Glance
            ↓
Inventory & Warehouse
```

The project demonstrates practical skills in:

- Business problem understanding
- Dimensional data modeling
- DAX measure development
- Power BI dashboard design
- Operational analytics
- Exception analysis
- SLA analysis
- Inventory-related analysis
- Drill-through reporting
