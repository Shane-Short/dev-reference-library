Issue Summary

A Power BI dataset failed to refresh in the Power BI Service (workspace), while the same report refreshed successfully in Power BI Desktop.
The Service error reported a semantic model processing failure caused by a data type mismatch on the column:

NOWDATA[MACHINE_EXIT_FACTORY_DATE]

The error indicated Power BI Service was unable to convert a value from a string (VT_BSTR) to a date (VT_DATE), which caused the entire refresh to be cancelled.

⸻

Root Cause

The column MACHINE_EXIT_FACTORY_DATE originates from Snowflake and contains values formatted as:

YYYY-MM-DD 00:00:00.000

While these values visually resemble dates, they are effectively datetime strings or datetime values with a time component.

In Power Query, an automatic “Changed Type” step was forcing this column to date.
This worked in Power BI Desktop, but Power BI Service is stricter during semantic model processing:
	•	If any row contains an empty value, unexpected text, or a value that cannot be converted cleanly to date,
	•	the Service refresh fails entirely (even if most rows are valid).

Desktop is more forgiving due to differences in locale handling, evaluation order, and query execution behavior.

⸻

Why Desktop Worked but Service Failed
	•	Power BI Desktop uses a more tolerant local execution context.
	•	Power BI Service enforces strict type validation during dataset refresh.
	•	A single invalid or non-convertible value in a forced type conversion causes the Service refresh to fail.
	•	This difference exposed a latent data quality / typing issue that Desktop did not surface.

⸻

Resolution Implemented

The fix was implemented in Power Query (Transform Data) to defensively sanitize the column before enforcing a data type.

Key Changes
	1.	Removed MACHINE_EXIT_FACTORY_DATE from the auto-generated Changed Type step.
	2.	Added an explicit cleaning step that:
	•	Converts values to text safely
	•	Trims whitespace
	•	Extracts the date portion (YYYY-MM-DD)
	•	Uses try … otherwise null to prevent refresh failures
	3.	Replaced the original column with the cleaned version and typed it as Date.

Result
	•	Invalid or unexpected values are converted to null instead of causing a failure.
	•	The dataset refresh now succeeds consistently in Power BI Service.
	•	Existing visuals, relationships, and measures were preserved (column name remained unchanged).

⸻

Final Outcome
	•	✅ Power BI Service refresh is stable and reliable
	•	✅ No functional or visual regressions
	•	✅ Root cause eliminated, not just masked
	•	✅ Future data anomalies in this column will no longer break refreshes

⸻

Preventive Recommendation

For any date/datetime fields sourced from Snowflake or other warehouses:
	•	Avoid relying solely on auto-generated Changed Type steps
	•	Use explicit, defensive type conversion (try … otherwise null)
	•	Treat Power BI Service as the source of truth for refresh validation, not Desktop

Optionally, the logic can be moved upstream into Snowflake (casting to DATE) for even stronger guarantees.

⸻

Business Impact
	•	Prevented ongoing refresh failures
	•	Reduced operational risk
	•	Improved robustness of enterprise reporting
	•	Eliminated a class of future refresh outages tied to data quality
