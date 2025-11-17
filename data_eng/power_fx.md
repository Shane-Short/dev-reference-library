Tool Health & PM Intelligence Dashboard – Proposed Structure & Data Architecture

(Draft for Review)

Below is the proposed architecture, page structure, and data model for the redesigned Parts Counter dashboard.
This design aligns with the SMART manufacturing vision, consolidates existing visuals, incorporates Erik’s POC, and adds new analytic capabilities to support Factory Management, Technical Support, and Reliability Engineering.

⸻

1. Data Model & Pipeline Architecture (Updated)

1.1 Primary Fact Tables

PM_All
	•	Reliable source for CEID, ENTITY, Process Node, Part, PM type, wafer count prior to reset, PM reset dates.
	•	Serves as the main event table for PM life calculations, PM classification, and chamber-level behavior.

PM_Flex
	•	Intel-supplied dataset containing maintenance thresholds, wafer deltas, downtime window classifications, reason detail, and ROI fields.
	•	Will be joined to PM_All for semantic enrichment.
	•	Key fields used include: Counter_Upper_Value, Median_Delta, Lower_IQR_Limit_Delta, Custom_Delta, PM_Reason_Deepdive, WO_Description, PM_Duration, downtime_type/class/subclass, and ROI metrics.

Prod_WW_Output_ES (Wafers Out)
	•	Provides wafer volume by WW, CEID, Entity, and Site.
	•	Used to normalize downtime and PM behavior to production context.
	•	Requires confirmation of completeness and availability.

Prod_Tool_List_Output_ES
	•	Critical for mapping CEID, Site, Chamber Type, and Process Node.
	•	Will also be used to derive Altair (GTAca) vs Non-Altair classification.

⸻

1.2 Altair (GTAca) vs Non-Altair Classification (New Requirement)

A dedicated classification field will be added to the model to differentiate between Altair (GTAca) and Non-Altair tools.
	•	Derived from the 3-digit CEID (GTA) + the module designation (“CA”)
	•	Exposed as a global slicer available on all relevant pages
	•	Integrated into CEID and Site comparison analyses
	•	Supports deeper part/PM behavior investigation driven by Altair differences
	•	If any existing dataset already contains this field, we will use it; otherwise, this will be created in the transformation layer.

⸻

1.3 Reference & Dimension Tables
	•	Tool/Chamber dimension (from Prod_Entity_Output_ES + Tool List).
	•	Date dimension (WW calendar).
	•	Part dimension (from PM_All and PM_Flex).
	•	Process Node dimension.
	•	Part_Price_Customer (pending confirmation).
	•	Part_Cost_Internal (pending confirmation).
	•	These two cost tables are required for warranty and spend analytics and must be validated before development.

⸻

1.4 Candidate for Deprecation

PM_Reset
	•	Appears to offer less detail compared to PM_All.
	•	Will be reviewed to confirm whether any unique fields require retention.

⸻

1.5 Chronic Tool Flag (New)

Algorithmically created metric based on:
	•	Unscheduled PM volume
	•	Variance of PM life vs Median_Delta
	•	PM stability
	•	Unscheduled downtime rate

Will be used for prioritization on the Chronic Tools page.

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
