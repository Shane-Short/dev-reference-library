-- CORRECTED Query 1
SELECT TOP 10
    pf.FACILITY,
    pf.ENTITY,
    pf.FACILITY + '_' + REPLACE(pf.ENTITY, '_PC', '_PM') AS temp_facility_entity,
    ws.FAB_ENTITY,
    ws.wafers_produced,
    COUNT(*) AS pm_event_count
FROM pm_flex_enriched pf
INNER JOIN vw_Wafer_Summary_3yr ws
    ON pf.FACILITY + '_' + REPLACE(pf.ENTITY, '_PC', '_PM') = ws.FAB_ENTITY
    AND pf.YEARWW = ws.WW
WHERE pf.YEARWW >= '2024WW01'
GROUP BY pf.FACILITY, pf.ENTITY, ws.FAB_ENTITY, ws.wafers_produced
ORDER BY ws.wafers_produced DESC;


-- CORRECTED Query 2
SELECT TOP 20
    ws.FAB_ENTITY,
    ws.WW,
    ws.wafers_produced,
    ws.parts_replaced_count,
    ws.max_part_count_this_week,
    (SELECT MAX(Part_Count) 
     FROM Prod_PM_all 
     WHERE FAB_ENTITY = ws.FAB_ENTITY 
       AND WW = ws.WW) AS raw_max_count
FROM vw_Wafer_Summary_3yr ws
WHERE ws.WW >= '2025WW40'  -- Last few weeks
ORDER BY ws.wafers_produced DESC;




-- =============================================
-- Create Wafer Summary View (3-Year Retention) - CORRECTED
-- =============================================
-- PURPOSE: Aggregate wafer production data from Prod_PM_all
-- FIX: Prevents double-counting when multiple parts replaced in same period
-- Logic: MAX(Part_Count) per tool per week (not SUM across parts)
-- Database: Parts_Counter_Production
-- Author: Shane (Data Engineer)
-- Created: 2025-12-09
-- Updated: 2025-12-09 - Fixed double-counting issue
-- =============================================

USE Parts_Counter_Production;
GO

-- =============================================
-- Drop existing view if it exists
-- =============================================
IF OBJECT_ID('dbo.vw_Wafer_Summary_3yr', 'V') IS NOT NULL
BEGIN
    PRINT 'Dropping existing view dbo.vw_Wafer_Summary_3yr...';
    DROP VIEW dbo.vw_Wafer_Summary_3yr;
    PRINT 'View dropped successfully.';
END
GO

-- =============================================
-- Create Corrected View: vw_Wafer_Summary_3yr
-- =============================================
PRINT 'Creating CORRECTED view dbo.vw_Wafer_Summary_3yr...';
GO

CREATE VIEW dbo.vw_Wafer_Summary_3yr AS
WITH WeeklyToolData AS (
    -- =============================================
    -- Step 1: Get max part count per tool per week
    -- This represents total wafers processed by tool during that week
    -- =============================================
    SELECT 
        FAB_ENTITY,
        ENTITY,
        CEID,
        WW,
        -- MAX part count = cumulative wafers through the tool
        MAX(Part_Count) AS max_part_count_this_week,
        -- Count how many parts were replaced this week
        COUNT(DISTINCT Part) AS parts_replaced_count,
        -- Most recent reset date this week
        MAX(Reset_Date) AS last_reset_date,
        -- First reset date this week (for reference)
        MIN(Reset_Date) AS first_reset_date
    FROM dbo.Prod_PM_all
    WHERE 
        -- Filter to last 3 years only
        Reset_Date >= DATEADD(YEAR, -3, GETDATE())
        -- Exclude NULL part counts
        AND Part_Count IS NOT NULL
        -- Exclude NULL work weeks
        AND WW IS NOT NULL
        -- Exclude NULL FAB_ENTITY (data quality issue)
        AND FAB_ENTITY IS NOT NULL
    GROUP BY FAB_ENTITY, ENTITY, CEID, WW
),
PreviousWeekData AS (
    -- =============================================
    -- Step 2: Get previous week's max part count
    -- =============================================
    SELECT 
        FAB_ENTITY,
        WW,
        max_part_count_this_week,
        -- Get previous week's part count for same tool
        LAG(max_part_count_this_week, 1, 0) OVER (
            PARTITION BY FAB_ENTITY 
            ORDER BY WW
        ) AS prev_week_max_count,
        parts_replaced_count,
        last_reset_date,
        first_reset_date,
        ENTITY,
        CEID
    FROM WeeklyToolData
)
SELECT 
    -- =============================================
    -- Step 3: Calculate wafers produced this week
    -- =============================================
    FAB_ENTITY,
    ENTITY,
    CEID,
    WW,
    
    -- Wafers produced = current max - previous max
    -- This avoids double-counting when multiple parts replaced
    CASE 
        WHEN max_part_count_this_week > prev_week_max_count 
            THEN max_part_count_this_week - prev_week_max_count
        WHEN max_part_count_this_week = prev_week_max_count 
            THEN 0  -- No new wafers
        ELSE 0  -- Handle counter resets (negative would indicate data issue)
    END AS wafers_produced,
    
    -- How many parts were replaced this week
    parts_replaced_count,
    
    -- Most recent reset date this week
    last_reset_date,
    
    -- First reset date this week (for validation)
    first_reset_date,
    
    -- Current cumulative part count
    max_part_count_this_week,
    
    -- Previous week's cumulative part count (for debugging)
    prev_week_max_count

FROM PreviousWeekData;
GO

-- =============================================
-- Verify view creation
-- =============================================
IF OBJECT_ID('dbo.vw_Wafer_Summary_3yr', 'V') IS NOT NULL
BEGIN
    PRINT '';
    PRINT '✓ CORRECTED view dbo.vw_Wafer_Summary_3yr created successfully!';
    PRINT '';
    
    DECLARE @RowCount INT;
    SELECT @RowCount = COUNT(*) FROM dbo.vw_Wafer_Summary_3yr;
    PRINT 'Total rows in view: ' + CAST(@RowCount AS VARCHAR(20));
    PRINT '';
END
ELSE
BEGIN
    PRINT '';
    PRINT '✗ ERROR: View creation failed!';
    PRINT '';
END
GO

-- =============================================
-- Sample data with new logic
-- =============================================
PRINT 'Sample data (top 20 rows by wafers_produced):';
PRINT '------------------------------------------------------------';

SELECT TOP 20
    FAB_ENTITY,
    ENTITY,
    CEID,
    WW,
    wafers_produced,
    parts_replaced_count,
    max_part_count_this_week,
    prev_week_max_count,
    last_reset_date
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced > 0  -- Only show weeks with production
ORDER BY wafers_produced DESC;
GO

-- =============================================
-- Validation: Compare old vs new logic
-- =============================================
PRINT '';
PRINT 'Validation: Spot-check a tool with multiple part replacements';
PRINT '------------------------------------------------------------';

-- Find a tool that had multiple parts replaced in one week
SELECT TOP 1
    FAB_ENTITY,
    WW,
    parts_replaced_count,
    wafers_produced,
    max_part_count_this_week,
    prev_week_max_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE parts_replaced_count > 1  -- Multiple parts replaced
  AND wafers_produced > 0
ORDER BY parts_replaced_count DESC;
GO

-- =============================================
-- Data quality checks
-- =============================================
PRINT '';
PRINT 'Data Quality Checks (CORRECTED):';
PRINT '------------------------------------------------------------';

SELECT 
    'NULL FAB_ENTITY count' AS check_name,
    COUNT(*) AS issue_count
FROM dbo.vw_Wafer_Summary_3yr
WHERE FAB_ENTITY IS NULL

UNION ALL

SELECT 
    'NULL WW count',
    COUNT(*)
FROM dbo.vw_Wafer_Summary_3yr
WHERE WW IS NULL

UNION ALL

SELECT 
    'Zero wafers produced count',
    COUNT(*)
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced = 0

UNION ALL

SELECT 
    'Negative wafers produced count',
    COUNT(*)
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced < 0

UNION ALL

SELECT 
    'Unrealistic wafers (>100M per week)',
    COUNT(*)
FROM dbo.vw_Wafer_Summary_3yr
WHERE wafers_produced > 100000000;  -- Flag suspiciously high values

GO

-- =============================================
-- Summary statistics (CORRECTED)
-- =============================================
PRINT '';
PRINT 'Summary Statistics (CORRECTED LOGIC):';
PRINT '------------------------------------------------------------';

SELECT 
    'Wafer Production Stats' AS category,
    MIN(wafers_produced) AS min_value,
    MAX(wafers_produced) AS max_value,
    AVG(wafers_produced) AS avg_value,
    SUM(wafers_produced) AS total_value,
    COUNT(*) AS row_count
FROM dbo.vw_Wafer_Summary_3yr

UNION ALL

SELECT 
    'Parts Replaced Stats',
    MIN(parts_replaced_count),
    MAX(parts_replaced_count),
    AVG(parts_replaced_count),
    SUM(parts_replaced_count),
    COUNT(*)
FROM dbo.vw_Wafer_Summary_3yr;

GO

-- =============================================
-- Validation query: Compare to raw data
-- =============================================
PRINT '';
PRINT 'Validation Query: Compare view output to raw Prod_PM_all';
PRINT '------------------------------------------------------------';
PRINT 'Checking one tool across multiple weeks...';

-- Pick a tool and show how wafer counts progress week-by-week
SELECT TOP 10
    ws.FAB_ENTITY,
    ws.WW,
    ws.wafers_produced,
    ws.max_part_count_this_week,
    ws.prev_week_max_count,
    ws.parts_replaced_count,
    -- Calculate cumulative wafers over time
    SUM(ws.wafers_produced) OVER (
        PARTITION BY ws.FAB_ENTITY 
        ORDER BY ws.WW 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_wafers
FROM dbo.vw_Wafer_Summary_3yr ws
WHERE ws.FAB_ENTITY = (
    -- Pick the tool with most activity
    SELECT TOP 1 FAB_ENTITY 
    FROM dbo.vw_Wafer_Summary_3yr 
    GROUP BY FAB_ENTITY 
    ORDER BY SUM(wafers_produced) DESC
)
ORDER BY ws.WW DESC;

GO

PRINT '';
PRINT '====================================================================';
PRINT 'CORRECTED View Creation Complete!';
PRINT '====================================================================';
PRINT '';
PRINT 'KEY CHANGES FROM PREVIOUS VERSION:';
PRINT '  - Uses MAX(Part_Count) per tool per week (not SUM across parts)';
PRINT '  - Prevents double-counting when multiple parts replaced';
PRINT '  - Calculates: current_max - previous_max = wafers this week';
PRINT '';
PRINT 'INTERPRETATION:';
PRINT '  - wafers_produced = wafers processed THIS WEEK by this tool';
PRINT '  - max_part_count_this_week = cumulative wafer count through tool';
PRINT '  - parts_replaced_count = how many parts were replaced this week';
PRINT '';
PRINT 'Next Steps:';
PRINT '  1. Compare results to previous version';
PRINT '  2. Validate wafer counts make sense (should be lower/more accurate)';
PRINT '  3. Run validation queries to spot-check accuracy';
PRINT '';
GO
