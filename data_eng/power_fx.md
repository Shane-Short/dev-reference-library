Tool Health & PM Intelligence Dashboard – Proposed Structure & Data Architecture

(Draft for Review)

Below is the proposed architecture, page structure, and data model for the redesigned Parts Counter dashboard.
This design aligns with the SMART manufacturing vision, consolidates existing visuals, incorporates Erik’s POC, and adds new analytic capabilities to support Factory Management, Technical Support, and Reliability Engineering.

⸻

1. Data Model & Pipeline Architecture

1.1 Primary Fact Tables

PM_All
	•	Reliable source for CEID, ENTITY, Process Node, Part, PM type, wafer count prior to reset, PM reset dates.
	•	This will serve as the primary event table for PM life calculations, PM classification, trend analysis, and chamber-level behavior.

PM_Flex
	•	Intel-supplied dataset containing maintenance thresholds, wafer deltas, downtime windows, PM reason classifications, and ROI fields.
	•	To be joined to PM_All for deeper semantic insight.
	•	Key fields used: Counter_Upper_Value, Median_Delta, Lower_IQR_Limit_Delta, Custom_Delta, Met_Upper_Limit, PM_Cycle_Utilization, reliable_upper_limit_insight, PM_Reason_Deepdive, WO_Description, Down_Window_Duration, PM_Duration, downtime_type/class/subclass, part_cost_per_pm, part_cost_saving_roi, mts_needed_laborhour_per_pm, etc.

Prod_WW_Output_ES (Wafers Out)
	•	Provides production context by WW, CEID, ENTITY, and Process Node.
	•	Used for normalizing downtime and PM rates.
	•	If this table is not currently populated or reliable, we will need confirmation.

Prod_Tool_List_Output_ES
	•	Used for mapping: CEID, site/fab, chamber type, process node.
	•	Critical for all site and CEID-level visuals.

⸻

1.2 Reference & Dimension Tables
	•	Tool/Chamber dimension (from Prod_Entity_Output_ES and Tool List).
	•	Date dimension (WW, Year-Month, Calendar).
	•	Part dimension (from PM_All + PM_Flex).
	•	Process Node dimension (from PM_Flex + Tool List).
	•	Cost tables (customer price + internal cost) – These do not exist yet and will require confirmation from the appropriate groups.

Once available, these will enable part cost analytics and warranty spend analysis.

⸻

1.3 Tables That May Be Deprecated or Limited Use

PM_Reset
	•	PM_All currently appears to contain more reliable event-level details than PM_Reset.
	•	We will validate whether PM_Reset contributes unique information; if not, it may be collapsed into PM_All logic.

⸻

1.4 Chronic Tool Definition (New)

Chambers will be algorithmically classified as “Chronic” based on:
	•	Unscheduled PM count (e.g., last 26 weeks)
	•	PM life variance relative to Median_Delta
	•	Unscheduled downtime rate
	•	Stability of PM intervals

This will become a standardized metric for reliability conversations.

⸻

2. Proposed Dashboard Page Structure

All existing visuals will be integrated where appropriate.
POC elements from Erik will also be merged into Pages 1–4.
New analytic pages have been added to support proactive and predictive maintenance.

⸻

Page 1 — Executive Overview

Use Case:
Quickly assess overall fleet health: scheduled vs unscheduled downtime, PM schedule adherence, and site/CEID trends.
Provides leadership with a consolidated health status of the entire etch fleet.

Visuals:
	•	KPI tiles
	•	Total PM events
	•	% Unscheduled
	•	Average PM Life vs Target (Median_Delta)
	•	Total PM-related downtime hours
	•	Unscheduled vs Scheduled downtime breakdown (from POC)
	•	Downtime subclass distribution
	•	Unscheduled Down Classifications by Site (from POC)
	•	Unscheduled Down Classifications by CEID (from POC)
	•	4-week rolling PM trend (existing PM A Trend visual)

⸻

Page 2 — Downtime Classification & Cause Analysis

(Expands on POC Overview)

Use Case:
Identify and quantify root causes of downtime, including unscheduled events, outlier downtime windows, and PM-related classifications.
Supports RCA, escalation prevention, and proactive tool health monitoring.

Visuals:
	•	Downtime Class/Subclass trend over time
	•	Pareto: Downtime_Subclass_Details
	•	Pareto: PM_Reason_Deepdive
	•	Optional: WO_Description keyword grouping (requires validation if feasible)

⸻

Page 3 — Site Comparison & Production Context

(Uses existing Wafer Starts pages + POC Site Investigation)

Use Case:
Compare site performance, PM timing, downtime normalization, and wafer volume to identify operational gaps.
Allows factory managers and regional leaders to benchmark against peers.

Visuals:
	•	Wafer Starts by Site (existing visual)
	•	Unscheduled PM rate by Site (from POC)
	•	Average PM Life vs Target by Site
	•	Downtime hours per 1,000 wafers (Wafers Out table required)
	•	If Wafers Out is incomplete for some time periods, we need confirmation from the data owners.

⸻

Page 4 — CEID / Module Investigation

(POC CEID Investigation + existing Part Count Trend)

Use Case:
Provide CEID owners and subject matter experts visibility into performance across their fleet.
Focus on wafer-based PM compliance, scatter behavior, and chamber variability.

Visuals:
	•	CEID selector with summary KPIs
	•	Scatter plot: wafer count at PM vs date, with upper/lower thresholds (POC)
	•	Part Replacement Count trend (existing Part Count Trend visual)
	•	Chamber ranking (Entity-level table with PM frequency, avg life, unscheduled rate)

⸻

Page 5 — PM Life & Part Investigation

(Enhanced version of existing PM A Investigation)

Use Case:
Drill down into a specific part or PM type to evaluate PM life consistency, outlier chambers, and replacement behavior.
Supports part-level investigations, PM interval optimization, and predictive modeling.

Visuals:
	•	Part selector + CEID filter
	•	Mean Reset Count by Chamber (existing)
	•	Time-series wafer counter “saw-tooth” trend (existing)
	•	Boxplot or distribution of PM life
	•	Event table (existing WW/Entity/Part/Reset Count + early/overdue flags)

⸻

Page 6 — Unscheduled Downtime & Chronic Tools

(New)

Use Case:
Identify the highest-risk chambers, understand what makes them chronic, and prioritize engineering resources toward the most impactful issues.

Visuals:
	•	Chronic Tools table
	•	Scatter plot: PM Life variance vs Unscheduled PM volume
	•	Pareto of unscheduled reasons (filtered to chronic tools)
	•	Drillthrough: PM_Flex row-level details (WO description, reason codes, wafer deltas)

⸻

Page 7 — Cost & Warranty Impact

(Requires confirmation of missing cost tables)

Use Case:
Translate maintenance and downtime issues into cost impact and identify opportunities for cost reduction.
This requires two datasets we do not yet have:
	1.	Part_Price_Customer (prices charged to customers)
	2.	Part_Cost_Internal (warranty/internal cost)

If these tables exist or can be provided, the following visuals will be enabled.

Visuals:
	•	Total parts spend vs warranty spend by Site/CEID
	•	Cost per PM and cost per wafer
	•	Top N parts by total spend
	•	Estimated savings from interval improvements (PM_Flex ROI fields)
	•	Requires validation of ROI fields in PM_Flex to ensure definitions are correct.

⸻

Page 8 — Analyst / Power User Explorer

(New)

Use Case:
Provide advanced users (technical support, reliability engineering, analysts) a flexible environment to explore relationships without requiring new dashboard development.

Visuals:
	•	Configurable matrix with selectable metrics
	•	Ad-hoc scatter plot tool
	•	Distribution charts for PM life, downtime duration, etc.

⸻

3. Integration of Existing Visuals

Below is where each current component will be reused:
	•	PM A Trend page → Executive Overview (Page 1)
	•	PM A Investigation → PM Life & Part Investigation (Page 5)
	•	Part Count Trend → CEID Investigation (Page 4)
	•	Wafer Starts → Site Comparison (Page 3)
	•	Wafer Starts Site Comparison → Site Comparison (Page 3)
	•	POC Overview → Pages 1 and 2
	•	POC Site Investigation → Page 3
	•	POC CEID Investigation → Page 4

No valuable content is removed; it is reorganized into a cohesive, hierarchical dashboard structure.

⸻

4. Summary

This proposal transforms the existing Parts Counter dashboard into the Tool Health & PM Intelligence Dashboard, moving beyond simple part counts toward a fully contextualized maintenance, downtime, and predictive insights platform.

Key improvements include:
	•	Unified data model using PM_All as the backbone with PM_Flex enrichment
	•	Clear separation of fleet-level, site-level, CEID-level, part-level, and chamber-level insights
	•	Integration of scheduled vs unscheduled downtime analysis (POC)
	•	Predictive and proactive metrics such as PM interval variance, stability scoring, and chronic tool identification
	•	(Future) cost and warranty linkage once the required datasets are confirmed
	•	Structured user flow enabling executives, engineers, and analysts to each access the level of detail they need

Once approved, we can begin wiring the data model, building the necessary transformation logic, and constructing each page iteratively with stakeholder feedback.
