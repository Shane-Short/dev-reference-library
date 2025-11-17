Tool Health & PM Intelligence Dashboard – Proposed Structure & Data Architecture

(Draft for Review – Updated to Include All Notes & Requirements)

Below is the proposed architecture, page structure, and data model for the redesigned Parts Counter dashboard.
This design aligns with the SMART manufacturing vision, consolidates existing visuals, incorporates Erik’s POC, and adds new analytic capabilities to support Factory Management, Technical Support, and Reliability Engineering.

⸻

1. Data Model & Pipeline Architecture (Updated)

1.1 Primary Fact Tables

PM_All
	•	Reliable source for CEID, ENTITY, Process Node, Part, PM type, wafer count prior to reset, and reset dates.
	•	Will serve as the primary event table for PM life calculations, PM classification, trend analysis, and chamber-level behavior.

PM_Flex
	•	Intel-supplied dataset containing maintenance thresholds, wafer deltas, downtime window classifications, reason detail, and ROI fields.
	•	Will be joined to PM_All for semantic enrichment.
	•	Key fields used include: Counter_Upper_Value, Median_Delta, Lower_IQR_Limit_Delta, Custom_Delta, Met_Upper_Limit, PM_Cycle_Utilization, reliable_upper_limit_insight, PM_Reason_Deepdive, WO_Description, Down_Window_Duration, PM_Duration, downtime_type/class/subclass, and ROI metrics.

Prod_WW_Output_ES (Wafers Out)
	•	Provides wafer volume by WW, CEID, Entity, and Site.
	•	Used to normalize downtime and PM behavior to production context.
	•	Requires confirmation of completeness and availability.

Prod_Tool_List_Output_ES
	•	Critical for mapping CEID, Site, Chamber Type, and Process Node.
	•	Will be used to derive Altair (GTAca) vs Non-Altair classification.

⸻

1.2 Altair (GTAca) vs Non-Altair Classification (New Requirement)

A dedicated classification field will be added to differentiate between Altair (GTAca) and Non-Altair tools.
	•	Derived from the 3-digit CEID (GTA) + module designation (“CA”).
	•	Exposed as a global slicer available on all relevant pages.
	•	Integrated into CEID, site, part, and chamber-level analysis.
	•	If any existing dataset already contains this field, we will reuse it; otherwise, we will derive it via transformation logic.

⸻

1.3 Reference & Dimension Tables
	•	Tool/Chamber dimension (from Prod_Entity_Output_ES + Tool List).
	•	Date dimension (WW calendar).
	•	Part dimension (from PM_All and PM_Flex).
	•	Process Node dimension.
	•	Part_Price_Customer (pending confirmation; required for cost analysis)
	•	Part_Cost_Internal (pending confirmation; required for warranty analysis)

These cost tables must be confirmed before development of the cost & warranty page.

⸻

1.4 Candidate for Deprecation

PM_Reset
	•	PM_All appears to offer richer and more reliable event-level details.
	•	We will review PM_Reset to confirm whether any unique fields require retention; otherwise, logic may be consolidated.

⸻

1.5 Chronic Tool Flag (New)

Algorithmically created metric based on:
	•	Unscheduled PM count
	•	Variance of PM life vs Median_Delta
	•	PM stability
	•	Unscheduled downtime rate

Used on the Chronic Tools page for prioritization.

⸻

2. Additional Requirements from Meeting Notes (Explicit Integration)

The following items were explicitly requested in earlier design meetings and have been incorporated into the dashboard plan:

2.1 Fix Wafer Starts Drill-Down Logic

Revive and correct the Wafer Starts and Site Comparison visuals to ensure WW → Month → Day drill-down operates properly.
The existing implementation was noted as broken.

2.2 Restore Raw Outs Visualization

Restore and correct the “Raw Outs” visualization to show wafers per chamber per week.
This visual is currently unreliable and needs repair to support chamber-level performance evaluation.

2.3 Include the Four-Category Overview Requested by Leadership

The Executive Overview page explicitly includes:
	1.	Unscheduled down classifications by site and type
	2.	Scheduled vs unscheduled PMs
	3.	Chronic issues by site
	4.	CEID-level PM and downtime breakdown by month and site

2.4 ES Ipro App Deployment

Final production deployment will be through the ES Ipro App to ensure accessibility for FSEs, Technical Support, and Factory Management involved in tool uptime and downtime analysis.

⸻

3. Proposed Dashboard Page Structure

All existing visuals are integrated where appropriate.
POC elements from Erik are merged into Pages 1–4.
New analytic pages support proactive and predictive maintenance.

⸻

Page 1 — Executive Overview

Use Case:
Assess overall fleet health: scheduled vs unscheduled downtime, PM schedule adherence, site/CEID differences, and high-level chronic patterns.

Visuals:
	•	KPI Tiles
	•	Total PM events
	•	% Unscheduled
	•	Average PM Life vs Target
	•	Total PM-related downtime hours
	•	Unscheduled vs Scheduled downtime breakdown (POC)
	•	Downtime subclass distribution
	•	Unscheduled down classifications by Site (POC)
	•	Unscheduled down classifications by CEID (POC)
	•	4-week rolling PM trend (existing PM A Trend)
	•	Altair vs Non-Altair slicer
	•	Four-category overview block requested by leadership

⸻

Page 2 — Downtime Classification & Cause Analysis

(Expanded POC Overview)

Use Case:
Identify root causes of downtime, unscheduled maintenance, outlier windows, and recurring failure patterns.
Supports RCA and proactive monitoring.

Visuals:
	•	Downtime Class/Subclass trend over time
	•	Pareto: Downtime_Subclass_Details
	•	Pareto: PM_Reason_Deepdive
	•	Optional keyword grouping of WO_Description
	•	Altair vs Non-Altair filter

⸻

Page 3 — Site Comparison & Production Context

(Existing Wafer Starts + POC Site Investigation)

Use Case:
Compare sites on PM timing, downtime classification, and fleet stability, normalized to wafer starts.
Allows benchmarking across factories.

Visuals:
	•	Wafer Starts by Site (existing, now with corrected drill-down)
	•	Unscheduled PM rate by Site (POC)
	•	Avg PM Life vs Target by Site
	•	Downtime hours per 1,000 wafers (requires Wafers Out completeness confirmation)
	•	Raw Outs (wafers per chamber per week; restored visual)
	•	Altair vs Non-Altair filter

⸻

Page 4 — CEID / Module Investigation

(POC CEID Investigation + existing Part Count Trend)

Use Case:
Equip CEID/module owners with visibility into chamber-level PM behavior, stability, wafer thresholds, and part usage patterns.

Visuals:
	•	CEID selector with summary KPIs
	•	Threshold scatter (Upper/Lower limits + PM events; POC)
	•	Part Replacement Count trend (existing Part Count Trend)
	•	Chamber ranking table
	•	Altair vs Non-Altair filter

⸻

Page 5 — PM Life & Part Investigation

(Enhanced existing PM A Investigation)

Use Case:
Enable targeted investigation for a specific part or PM type to analyze PM consistency, chamber variability, and outliers.

Visuals:
	•	Part selector + CEID filter
	•	Mean Reset Count by Chamber (existing)
	•	Wafer counter “saw-tooth” visual (existing)
	•	Distribution/Boxplot of PM Life
	•	Enhanced event table with early/on-time/overdue flags
	•	Altair vs Non-Altair filter

⸻

Page 6 — Unscheduled Downtime & Chronic Tools

(New)

Use Case:
Identify high-risk chambers and root causes of chronic behavior to prioritize engineering resources.

Visuals:
	•	Chronic Tools table
	•	Scatter: PM Life variance vs Unscheduled PM count
	•	Pareto of unscheduled reasons (filtered to chronic tools)
	•	Drillthrough to PM_Flex event detail
	•	Altair vs Non-Altair filter

⸻

Page 7 — Cost & Warranty Impact

(Requires confirmation of cost tables)

Use Case:
Translate maintenance behavior into cost and warranty impact.
Requires confirmation of existence of two tables: Part_Price_Customer and Part_Cost_Internal.

Visuals:
	•	Total parts spend vs warranty spend by Site/CEID
	•	Cost per PM / Cost per wafer
	•	Top N parts by spend
	•	Estimated savings (PM_Flex ROI fields)
	•	Altair vs Non-Altair filter

⸻

Page 8 — Analyst / Power User Explorer

(New)

Use Case:
Provide analysts and reliability engineers with a flexible environment for custom queries and exploratory analysis.

Visuals:
	•	Configurable metric matrix
	•	Ad-hoc scatter tool
	•	Distribution charts for PM Life, downtime duration, etc.
	•	Altair vs Non-Altair filter

⸻

4. Integration of Existing Visuals
	•	PM A Trend → Page 1
	•	PM A Investigation → Page 5
	•	Part Count Trend → Page 4
	•	Wafer Starts → Page 3
	•	Wafer Starts Site Comparison → Page 3
	•	POC Overview → Pages 1 and 2
	•	POC Site Investigation → Page 3
	•	POC CEID Investigation → Page 4

Nothing is removed; it is reorganized into a cohesive, hierarchical structure.

⸻

5. Summary

This redesign transforms Parts Counter into the Tool Health & PM Intelligence Dashboard, delivering:
	•	Unified data model with PM_All backbone and PM_Flex semantic enrichment
	•	Accurate PM schedule adherence analysis
	•	Clear fleet, site, CEID, part, and chamber-level insights
	•	Unscheduled vs scheduled downtime visibility
	•	Chronic tool detection
	•	Predictive behavior insights (via PM variance and drift)
	•	Cost and warranty analytics (if required tables are available)
	•	Altair vs Non-Altair segmentation
	•	ES Ipro App deployment for maximum accessibility
