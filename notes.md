Root Cause

The Power BI dataset failed to refresh in the Power BI Service due to a data type conversion error on the column MACHINE_EXIT_FACTORY_DATE.

Although the values visually appeared to be valid dates, the source data from Snowflake contained datetime values formatted as YYYY-MM-DD 00:00:00.000. During refresh, Power BI Service attempted to force this column into a Date type via an automatic “Changed Type” step. If even a single row could not be converted cleanly, the Service aborted the semantic model processing.

Power BI Desktop did not surface this issue due to differences in local execution, locale handling, and tolerance for type coercion, which masked the underlying data quality problem.

⸻

The reason:

Power BI Service enforces stricter schema validation than Power BI Desktop.
The auto-generated Changed Type step forced a hard conversion from text/datetime to date without accounting for nulls, blanks, or non-standard values. This caused the Service refresh to fail when encountering any row that did not meet strict date conversion rules, even though most rows were valid.

⸻

What I Changed / Fix Implemented

I removed the forced type conversion for MACHINE_EXIT_FACTORY_DATE from the existing Changed Type step and introduced a defensive transformation step earlier in Power Query.

The new step:
	•	Safely converts the value to text
	•	Trims whitespace
	•	Extracts only the date portion (YYYY-MM-DD) from datetime values
	•	Uses a fail-safe conversion that returns null instead of throwing an error

After cleaning the data, the original column was replaced with the sanitized version and explicitly typed as a Date, ensuring compatibility with both Power BI Desktop and Power BI Service.

⸻

Result / Impact
	•	Power BI Service refresh now completes successfully and consistently
	•	The dataset is resilient to malformed or unexpected values in the date column
	•	No report visuals, measures, or relationships were impacted
	•	Eliminated a hidden failure mode that could recur with future data changes
	•	Reduced operational risk and manual intervention

⸻

Why this is scalable

This approach:
	•	Prevents single-row data issues from breaking entire dataset refreshes
	•	Works consistently across Desktop, Service, and gateway environments
	•	Can be reused as a standard pattern for all date/datetime fields
	•	Is resilient to upstream schema changes or data anomalies
	•	Aligns with enterprise best practices for defensive data modeling

This solution ensures long-term stability without relying on assumptions about data cleanliness or execution environment differences.
