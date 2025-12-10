-- =============================================
-- Create Wafer Summary View (3-Year Retention)
-- =============================================
-- Purpose: Aggregate wafer production data from Prod_PM_all
-- Reduces 81M rows to ~50K for Power BI performance
-- Database: Parts_Counter_Production
-- Author: Shane (Data Engineer)
-- Created: 2025-12-09
-- =============================================

USE Parts_Counter_Production;
GO

-- =============================================
-- Drop existing view if it exists (for re-creation)
-- =============================================
IF OBJECT_ID('dbo.vw_Wafer_Summary_3yr', 'V') IS NOT NULL
BEGIN
    PRINT 'Dropping existing view dbo.vw_Wafer_Summary_3yr...';
    DROP VIEW dbo.vw_Wafer_Summary_3yr;
    PRINT 'View dropped successfully.';
END
GO

-- =============================================
-- Create View: vw_Wafer_Summary_3yr
-- =============================================
PRINT 'Creating view dbo.vw_Wafer_Summary_3yr...';
GO

CREATE VIEW dbo.vw_Wafer_Summary_3yr AS
WITH RankedParts AS (
    -- =============================================
    -- Step 1: Get all part replacements from last 3 years
    -- Calculate previous part count using LAG window function
    -- =============================================
    SELECT 
        FAB_ENTITY,
        ENTITY,
        CEID,
        WW,
        Part,
        Reset_Date,
        Part_Count,
        -- Get previous part count for same tool+part to calculate wafers since reset
        LAG(Part_Count, 1, 0) OVER (
            PARTITION BY FAB_ENTITY, Part 
            ORDER BY Reset_Date
        ) AS prev_part_count,
        -- Row number for deduplication (in case of duplicate entries)
        ROW_NUMBER() OVER (
            PARTITION BY FAB_ENTITY, Part, Reset_Date 
            ORDER BY Reset_Date DESC
        ) AS rn
    FROM dbo.Prod_PM_all
    WHERE 
        -- Filter to last 3 years only (matches Gold layer retention)
        Reset_Date >= DATEADD(YEAR, -3, GETDATE())
        -- Exclude NULL part counts (invalid data)
        AND Part_Count IS NOT NULL
        -- Exclude NULL work weeks (can't aggregate without WW)
        AND WW IS NOT NULL
)
SELECT 
    -- =============================================
    -- Step 2: Aggregate by tool and week
    -- =============================================
    FAB_ENTITY,                    -- Join key to pm_flex_enriched (after FACILITY_ENTITY created)
    ENTITY,                        -- Tool entity (e.g., 'GTA123_PM1')
    CEID,                          -- Tool family (e.g., 'GTAca')
    WW,                            -- Work week (e.g., '2025WW48')
    
    -- Calculate total wafers produced this week for this tool
    -- Logic: Sum of (current part count - previous part count) for all parts
    SUM(
        CASE 
            WHEN Part_Count > prev_part_count THEN Part_Count - prev_part_count
            WHEN Part_Count = prev_part_count THEN 0  -- No wafers if count same
            ELSE 0  -- Handle resets or negative values (data quality issue)
        END
    ) AS wafers_produced,
    
    -- Count how many unique parts were replaced this week
    COUNT(DISTINCT Part) AS parts_replaced_count,
    
    -- Most recent reset date this week
    MAX(Reset_Date) AS last_reset_date,
    
    -- Highest part count recorded this week (for reference)
    MAX(Part_Count) AS max_part_count_this_week
    
FROM RankedParts
WHERE rn = 1  -- Deduplicate if multiple entries for same tool+part+date
GROUP BY FAB_ENTITY, ENTITY, CEID, WW;
GO

-- =============================================
-- Verify view creation
-- =============================================
IF OBJECT_ID('dbo.vw_Wafer_Summary_3yr', 'V') IS NOT NULL
BEGIN
    PRINT '';
    PRINT '✓ View dbo.vw_Wafer_Summary_3yr created successfully!';
    PRINT '';
    
    -- Get row count (should be ~50K rows)
    DECLARE @RowCount INT;
    SELECT @RowCount = COUNT(*) FROM dbo.vw_Wafer_Summary_3yr;
    PRINT 'Total rows in view: ' + CAST(@RowCount AS VARCHAR(20));
    PRINT '';
    
    -- Show sample data
    PRINT 'Sample data (top 10 rows by last_reset_date):';
    PRINT '------------------------------------------------------------';
END
ELSE
BEGIN
    PRINT '';
    PRINT '✗ ERROR: View creation failed!';
    PRINT '';
END
GO

-- =============================================
-- Display sample data for validation
-- =============================================
SELECT TOP 10
    FAB_ENTITY,
    ENTITY,
    CEID,
    WW,
    wafers_produced,
    parts_replaced_count,
    last_reset_date,
    max_part_count_this_week
FROM dbo.vw_Wafer_Summary_3yr
ORDER BY last_reset_date DESC;
GO

-- =============================================
-- Performance validation query
-- =============================================
PRINT '';
PRINT 'Performance Test: Query execution time for view';
PRINT '------------------------------------------------------------';
SET STATISTICS TIME ON;

SELECT 
    COUNT(*) AS total_rows,
    SUM(wafers_produced) AS total_wafers,
    COUNT(DISTINCT FAB_ENTITY) AS unique_tools,
    COUNT(DISTINCT WW) AS unique_weeks
FROM dbo.vw_Wafer_Summary_3yr;

SET STATISTICS TIME OFF;
GO

-- =============================================
-- Data quality checks
-- =============================================
PRINT '';
PRINT 'Data Quality Checks:';
PRINT '------------------------------------------------------------';

-- Check for NULL values in critical columns
SELECT 
    'NULL FAB_ENTITY count' AS check_name,
    COUNT(*) AS issue_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE FAB_ENTITY IS NULL

UNION ALL

SELECT 
    'NULL WW count' AS check_name,
    COUNT(*) AS issue_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE WW IS NULL

UNION ALL

SELECT 
    'Zero wafers produced count' AS check_name,
    COUNT(*) AS issue_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced = 0

UNION ALL

SELECT 
    'Negative wafers produced count' AS check_name,
    COUNT(*) AS issue_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced < 0;

GO

-- =============================================
-- Summary statistics
-- =============================================
PRINT '';
PRINT 'Summary Statistics:';
PRINT '------------------------------------------------------------';

SELECT 
    'Wafer Production Stats' AS category,
    MIN(wafers_produced) AS min_value,
    MAX(wafers_produced) AS max_value,
    AVG(wafers_produced) AS avg_value,
    SUM(wafers_produced) AS total_value
FROM dbo.vw_Wafer_Summary_3yr

UNION ALL

SELECT 
    'Parts Replaced Stats' AS category,
    MIN(parts_replaced_count) AS min_value,
    MAX(parts_replaced_count) AS max_value,
    AVG(parts_replaced_count) AS avg_value,
    SUM(parts_replaced_count) AS total_value
FROM dbo.vw_Wafer_Summary_3yr;

GO

PRINT '';
PRINT '====================================================================';
PRINT 'View Creation Complete!';
PRINT '====================================================================';
PRINT '';
PRINT 'Next Steps:';
PRINT '  1. Review sample data above to validate wafer calculations';
PRINT '  2. Check data quality results - should show 0 issues';
PRINT '  3. If row count is ~50K, performance is good for Power BI';
PRINT '  4. Ready to import into Power BI as data source';
PRINT '';
PRINT 'View Name: dbo.vw_Wafer_Summary_3yr';
PRINT 'Database: Parts_Counter_Production';
PRINT '';
GO
