# Power BI Report Blueprint & UI Design Specification

This document details the layout, visual hierarchy, KPI cards, and interaction patterns for the 3-page Power BI Jira / ITSM Dashboard.

---

## Design System & Color Tokens

| Semantic Role | Hex Code | Usage |
| :--- | :--- | :--- |
| **Canvas Background** | `#F8FAFC` (Slate 50) | Clean off-white background |
| **Card / Container Background** | `#FFFFFF` | Visual containers with subtle shadow `rgba(0,0,0,0.04)` |
| **Primary Navy (Brand)** | `#0F172A` (Slate 900) | Headers, primary numbers, dark accents |
| **Accent Blue** | `#2563EB` (Blue 600) | Opened tickets, primary flow, active tabs |
| **Success Green** | `#16A34A` (Green 600) | Resolved tickets, SLA Met ($ \ge 95\% $) |
| **Warning Amber** | `#D97706` (Amber 600) | SLA At Risk ($ 85\% - 94.9\% $), Awaiting User |
| **Danger Red** | `#DC2626` (Red 600) | SLA Breached, P1 Critical Outages |

---

## Page 1: Executive ITSM & Cloud Ops Overview

**Target Audience**: Head of Infrastructure, VP of Engineering, SRE Directors.  
**Core Question Answered**: *Are our critical cloud services stable, are we meeting our operational SLAs, and is our team keeping pace with incoming ticket volume?*

### Layout Structure (1920 x 1080)

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ TOP BAR: Global Filters (Date Range Slicer | Priority Tier | Department / Domain | Support Tier)       │
├─────────────────┬──────────────────┬──────────────────┬──────────────────┬─────────────────────────────┤
│ KPI CARD 1      │ KPI CARD 2       │ KPI CARD 3       │ KPI CARD 4       │ KPI CARD 5                  │
│ Incidents Opened│ Incidents Resolve│ Net Inflow       │ SLA Compliance % │ MTTR (Business Hours)       │
│ 24,918          │ 23,362           │ +1,556           │ 63.4% [Status]   │ 42.9 hrs (Median: 6.5 hrs)  │
├─────────────────┴──────────────────┴──────────────────┴──────────────────┴─────────────────────────────┤
│ VISUAL 1: Ticket Inflow vs. Outflow Trend (Line & Clustered Column Chart)                              │
│ • X-Axis: Dim_Calendar[YearMonth]                                                                      │
│ • Columns: [Incidents Opened] (Blue), [Incidents Resolved] (Green)                                     │
│ • Line (Secondary Y): [Active Backlog (Point-in-Time)] (Navy)                                          │
├────────────────────────────────────────┬───────────────────────────────────────────────────────────────┤
│ VISUAL 2: SLA Compliance % by Priority │ VISUAL 3: Incidents by Technical Domain & Department          │
│ (100% Stacked Bar Chart)               │ (Treemap / Donut Chart)                                       │
│ • Y-Axis: Dim_Priority[PriorityName]   │ • Categories: Cloud & Infra, Database & Messaging, Network    │
│ • Values: [SLA Met Count] (Green) vs.  │ • Values: [Incidents Opened]                                  │
│   [SLA Breached Count] (Red)           │ • Tooltip: [MTTR Business Hours], [SLA Compliance %]          │
└────────────────────────────────────────┴───────────────────────────────────────────────────────────────┘
```

---

## Page 2: SLA Compliance & MTTR Efficiency Deep-Dive

**Target Audience**: SRE Team Leads, Incident Commanders, Operations Managers.  
**Core Question Answered**: *Where are SLA breaches concentrating, and how does our resolution time compare across priority tiers and support teams?*

### Layout Structure

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ TOP BAR: Focus Filters (Priority Selector: P1-Critical / P2-High | Assignment Group | Breach Status)   │
├────────────────────────────────────────┬───────────────────────────────────────────────────────────────┤
│ VISUAL 1: MTTR vs. Contracted SLA Target│ VISUAL 2: SLA Breach Drivers (Decomposition Tree)             │
│ (Clustered Bar Chart)                  │ • Root Metric: [SLA Breached Count]                           │
│ • Y-Axis: Dim_Priority[PriorityName]   │ • Drill Levels: Department -> Category -> Assignment Group    │
│ • Bars: [MTTR Business Hours] vs.      │ • Identifies root teams and components responsible for        │
│   [Target SLA Business Hours]          │   unfulfilled SLAs.                                           │
├────────────────────────────────────────┴───────────────────────────────────────────────────────────────┤
│ VISUAL 3: Operational Resolution Matrix (Matrix Table with Heatmap Bars)                              │
│ • Rows: Dim_AssignmentGroup[Department] -> Dim_AssignmentGroup[AssignmentGroupName]                    │
│ • Columns: Dim_Priority[PriorityCode] (P1, P2, P3, P4)                                                 │
│ • Values: [Incidents Opened], [SLA Compliance %] (Data Bar conditional formatting),                    │
│           [MTTR Business Hours], [Avg Reassignments per Incident]                                      │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Page 3: Bottleneck Analysis & Changelog Process Mining

**Target Audience**: DevOps Process Engineers, Continuous Improvement Leads, IT Service Managers.  
**Core Question Answered**: *Why are tickets delayed? Which intermediate states (`Awaiting User Info`, `Awaiting Problem`, `Active`) cause the longest bottlenecks, and where are tickets bouncing?*

### Layout Structure

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│ TOP BAR: State Filters | Bottleneck Flag Filter | Department Filter                                    │
├────────────────────────────────────────┬───────────────────────────────────────────────────────────────┤
│ VISUAL 1: Average Dwell Time by State  │ VISUAL 2: Reassignment Churn Distribution                     │
│ (Horizontal Bar Chart)                 │ (Histogram / Clustered Column)                                │
│ • Y-Axis: Dim_State[StateName]         │ • X-Axis: Fact_Incidents[ReassignmentCount] (0, 1, 2, 3, 4+) │
│ • Values: [Avg Duration in State       │ • Values: Count of Incidents                                  │
│   (Business Hours)]                    │ • Highlight: Incidents with >= 3 reassignments (High Churn)   │
├────────────────────────────────────────┴───────────────────────────────────────────────────────────────┤
│ VISUAL 3: State Transition Path & Bottleneck Matrix                                                    │
│ • Rows: Fact_Transitions[FromState]                                                                   │
│ • Columns: Fact_Transitions[ToState]                                                                   │
│ • Values: [Total Transitions] (Count), [Avg Duration in State (Business Hours)] (Heatmap)              │
│ • Highlights looping paths (e.g. Active -> Awaiting User Info -> Active)                               │
├────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ VISUAL 4: High-Churn Incident Incident Drill-Through Table                                             │
│ • Columns: IncidentKey | Priority | Department | Reassignments | MTTR (Hours) | MadeSLA? | FinalState  │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Power BI Loading Guide

1. Open **Power BI Desktop**.
2. Click **Get Data** $\rightarrow$ **Text/CSV** (or **Folder** pointing to `data/processed/`).
3. Import all 7 CSV files:
   * `Fact_Incidents.csv`
   * `Fact_Transitions.csv`
   * `Dim_Calendar.csv`
   * `Dim_Priority.csv`
   * `Dim_State.csv`
   * `Dim_AssignmentGroup.csv`
   * `Dim_Category.csv`
4. In the **Model View**, verify the relationships match `docs/DATA_MODEL.md`.
   * Ensure `Fact_Incidents[ResolvedDateKey]` $\rightarrow$ `Dim_Calendar[DateKey]` is set to **Inactive**.
   * Ensure `Fact_Incidents[ClosedDateKey]` $\rightarrow$ `Dim_Calendar[DateKey]` is set to **Inactive**.
5. Create a new measure table `_Measures` and paste the DAX formulas from `dax/measures.dax`.
6. Assemble the 3 report pages using the layouts specified above.
