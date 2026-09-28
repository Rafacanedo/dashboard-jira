# Data Model Architecture & Schema Documentation

## 1. Overview & Architecture Pattern

This data model follows a Kimball-style **Star Schema** with two central fact tables representing distinct business granularities and five supporting dimension tables:

1. **`Fact_Incidents`**: Ticket-level grain (1 row per unique incident). Captures opened/resolved/closed timestamps, SLA status, churn metrics, and end-to-end resolution durations.
2. **`Fact_Transitions`**: Event-level audit changelog grain (1 row per status change). Captures the lifecycle path, durations spent in each state, and operational bottlenecks.

```mermaid
erDiagram
    Dim_Calendar ||--o{ Fact_Incidents : "OpenedDateKey (Active)"
    Dim_Calendar ||--o{ Fact_Incidents : "ResolvedDateKey (Inactive)"
    Dim_Calendar ||--o{ Fact_Incidents : "ClosedDateKey (Inactive)"
    Dim_Calendar ||--o{ Fact_Transitions : "TransitionDateKey (Active)"

    Dim_Priority ||--o{ Fact_Incidents : "PriorityKey"
    Dim_State ||--o{ Fact_Incidents : "FinalStateKey"
    Dim_AssignmentGroup ||--o{ Fact_Incidents : "AssignmentGroupKey"
    Dim_Category ||--o{ Fact_Incidents : "CategoryKey"

    Dim_State ||--o{ Fact_Transitions : "ToStateKey"
    Dim_AssignmentGroup ||--o{ Fact_Transitions : "AssignmentGroupKey"
    Fact_Incidents ||--o{ Fact_Transitions : "IncidentKey"

    Fact_Incidents {
        string IncidentKey PK
        int OpenedDateKey FK
        datetime OpenedDateTime
        int ResolvedDateKey FK
        datetime ResolvedDateTime
        int ClosedDateKey FK
        datetime ClosedDateTime
        int PriorityKey FK
        int FinalStateKey FK
        string AssignmentGroupKey FK
        string CategoryKey FK
        int MadeSLA
        int BreachedSLA
        float MTTR_BusinessHours
        float LeadTime_BusinessHours
        int ReassignmentCount
        int ReopenCount
    }

    Fact_Transitions {
        int TransitionKey PK
        string IncidentKey FK
        int StepNumber
        string FromState
        string ToState
        int ToStateKey FK
        datetime TransitionDateTime
        int TransitionDateKey FK
        float Duration_BusinessHours
        int IsBottleneck
    }

    Dim_Calendar {
        int DateKey PK
        date Date
        int Year
        string YearQuarter
        string YearMonth
        string MonthName
        int DayOfWeek
        string DayName
        int IsBusinessDay
        int IsWeekend
    }

    Dim_Priority {
        int PriorityKey PK
        string PriorityCode
        string PriorityName
        string SeverityTier
        float TargetSLA_Hours
        float TargetSLA_BusinessHours
    }

    Dim_State {
        int StateKey PK
        string StateName
        string StateStage
        int IsActiveBacklog
        int IsTerminal
        int StateSortOrder
    }

    Dim_AssignmentGroup {
        string AssignmentGroupKey PK
        string AssignmentGroupName
        string Department
        string SupportTier
    }

    Dim_Category {
        string CategoryKey PK
        string CategoryName
        string SubcategoryName
        string Domain
    }
```

---

## 2. Relationship Specifications

| From Table | From Column | To Table | To Column | Cardinality | Cross Filter | State in Model | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `Fact_Incidents` | `OpenedDateKey` | `Dim_Calendar` | `DateKey` | Many to One (`*:1`) | Single | **Active** | Primary date context for tickets created. |
| `Fact_Incidents` | `ResolvedDateKey` | `Dim_Calendar` | `DateKey` | Many to One (`*:1`) | Single | **Inactive** | Activated via `USERELATIONSHIP` for resolution trends. |
| `Fact_Incidents` | `ClosedDateKey` | `Dim_Calendar` | `DateKey` | Many to One (`*:1`) | Single | **Inactive** | Activated via `USERELATIONSHIP` for closure/lead time. |
| `Fact_Incidents` | `PriorityKey` | `Dim_Priority` | `PriorityKey` | Many to One (`*:1`) | Single | **Active** | Slice by Priority (P1–P4) & contracted SLA targets. |
| `Fact_Incidents` | `FinalStateKey` | `Dim_State` | `StateKey` | Many to One (`*:1`) | Single | **Active** | Filter by final status stage. |
| `Fact_Incidents` | `AssignmentGroupKey` | `Dim_AssignmentGroup` | `AssignmentGroupKey` | Many to One (`*:1`) | Single | **Active** | Department & support tier attribution. |
| `Fact_Incidents` | `CategoryKey` | `Dim_Category` | `CategoryKey` | Many to One (`*:1`) | Single | **Active** | Technical domain breakdown. |
| `Fact_Transitions` | `IncidentKey` | `Fact_Incidents` | `IncidentKey` | Many to One (`*:1`) | Single | **Inactive** | Eliminates ambiguous path loop with Dim_Calendar. Available for explicit DAX `USERELATIONSHIP`. |
| `Fact_Transitions` | `TransitionDateKey` | `Dim_Calendar` | `DateKey` | Many to One (`*:1`) | Single | **Active** | Audit timeline & transition date analysis. |
| `Fact_Transitions` | `ToStateKey` | `Dim_State` | `StateKey` | Many to One (`*:1`) | Single | **Active** | Dwell duration in each lifecycle state. |
| `Fact_Transitions` | `AssignmentGroupKey` | `Dim_AssignmentGroup` | `AssignmentGroupKey` | Many to One (`*:1`) | Single | **Active** | Team attribution at time of transition. |

---

## 3. Advanced Modeling Challenges Solved

### A. Role-Playing Date Dimensions (`USERELATIONSHIP`)
A single incident records three critical chronological milestones: `OpenedDateTime`, `ResolvedDateTime`, and `ClosedDateTime`. In a relational model, creating multiple active relationships between `Fact_Incidents` and `Dim_Calendar` causes relationship ambiguity. 

* **Solution**: Keep `OpenedDateKey` as the single active relationship. All measures tracking resolutions and closures explicitly invoke:
  ```dax
  USERELATIONSHIP('Fact_Incidents'[ResolvedDateKey], 'Dim_Calendar'[DateKey])
  ```
  This allows a single date slicer to drive both Inflow and Outflow simultaneously on the same visual.

### B. Point-in-Time Non-Additive Backlog
Standard Power BI sums cannot tell you how many tickets were open on March 15th, 2016. Adding daily open tickets across months would cause severe double-counting (non-additive metric).

* **Solution**: Semi-additive DAX logic evaluating whether each incident was opened on or before the boundary timestamp ($T_{\text{max}}$) and remained unresolved at that timestamp ($T_{\text{resolved}} > T_{\text{max}} \lor \text{ISBLANK}(T_{\text{resolved}})$):
  ```dax
  CALCULATE(
      COUNTROWS('Fact_Incidents'),
      FILTER(
          ALL('Fact_Incidents'),
          'Fact_Incidents'[OpenedDateTime] <= MaxSelectedDate + TIME(23, 59, 59) &&
          (
              ISBLANK('Fact_Incidents'[ResolvedDateTime]) ||
              'Fact_Incidents'[ResolvedDateTime] > MaxSelectedDate + TIME(23, 59, 59)
          )
      )
  )
  ```

### C. True Business-Hour Calculations
Incidents submitted at 4:30 PM on a Friday and resolved at 9:30 AM on Monday reflect **1 hour of working engineering time**, but **65 clock hours**. 
* The ETL pre-computes business hours ($M-F, 09:00 - 17:00$) at ingestion using vectorized calendar operations.
* Power BI displays both `MTTR_ClockHours` (for raw SLA uptime) and `MTTR_BusinessHours` (for engineering capacity efficiency).
