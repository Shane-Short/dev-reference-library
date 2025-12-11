-- =============================================
-- Create Part Performance View (3-Year Retention)
-- =============================================
-- PURPOSE: Track individual part performance and replacement patterns
-- Shows: How long parts last, which parts fail frequently, part reliability
-- Database: Parts_Counter_Production
-- Author: Shane (Data Engineer)
-- Created: 2025-12-09
-- =============================================

USE Parts_Counter_Production;
GO

-- =============================================
-- Drop existing view if it exists
-- =============================================
IF OBJECT_ID('dbo.vw_Part_Performance_3yr', 'V') IS NOT NULL
BEGIN
    PRINT 'Dropping existing view dbo.vw_Part_Performance_3yr...';
    DROP VIEW dbo.vw_Part_Performance_3yr;
    PRINT 'View dropped successfully.';
END
GO

-- =============================================
-- Create View: vw_Part_Performance_3yr
-- =============================================
PRINT 'Creating view dbo.vw_Part_Performance_3yr...';
GO

CREATE VIEW dbo.vw_Part_Performance_3yr AS
SELECT 
    -- Tool Identifiers
    FAB_ENTITY,
    ENTITY,
    CEID,
    
    -- Part Information
    Part,
    
    -- Replacement Event Details
    Reset_Date AS replacement_date,
    WW AS replacement_ww,
    Part_Count AS wafers_at_replacement,
    
    -- Previous Replacement for This Part (on this tool)
    LAG(Reset_Date) OVER (
        PARTITION BY FAB_ENTITY, Part 
        ORDER BY Reset_Date
    ) AS previous_replacement_date,
    
    LAG(Part_Count) OVER (
        PARTITION BY FAB_ENTITY, Part 
        ORDER BY Reset_Date
    ) AS wafers_at_previous_replacement,
    
    LAG(WW) OVER (
        PARTITION BY FAB_ENTITY, Part 
        ORDER BY Reset_Date
    ) AS previous_replacement_ww,
    
    -- Calculate Part Life Metrics
    -- Note: Part_Count represents cumulative wafers through THIS PART since it was installed
    Part_Count AS part_life_wafers,
    
    -- Days between replacements for this part
    DATEDIFF(DAY, 
        LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
        Reset_Date
    ) AS days_since_last_replacement,
    
    -- Weeks between replacements
    DATEDIFF(WEEK, 
        LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
        Reset_Date
    ) AS weeks_since_last_replacement,
    
    -- Categorize replacement frequency
    CASE 
        WHEN DATEDIFF(WEEK, 
            LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
            Reset_Date
        ) IS NULL THEN 'First Installation'
        WHEN DATEDIFF(WEEK, 
            LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
            Reset_Date
        ) < 4 THEN 'Very Frequent (<1 month)'
        WHEN DATEDIFF(WEEK, 
            LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
            Reset_Date
        ) < 8 THEN 'Frequent (1-2 months)'
        WHEN DATEDIFF(WEEK, 
            LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
            Reset_Date
        ) < 13 THEN 'Normal (2-3 months)'
        WHEN DATEDIFF(WEEK, 
            LAG(Reset_Date) OVER (PARTITION BY FAB_ENTITY, Part ORDER BY Reset_Date),
            Reset_Date
        ) < 26 THEN 'Infrequent (3-6 months)'
        ELSE 'Very Infrequent (>6 months)'
    END AS replacement_frequency_category,
    
    -- Part life quality indicator
    CASE 
        WHEN Part_Count < 5000 THEN 'Short Life (<5K wafers)'
        WHEN Part_Count < 10000 THEN 'Below Average (5K-10K)'
        WHEN Part_Count < 20000 THEN 'Average (10K-20K)'
        WHEN Part_Count < 30000 THEN 'Above Average (20K-30K)'
        ELSE 'Long Life (>30K wafers)'
    END AS part_life_category,
    
    -- Additional context columns from Prod_PM_all
    Process_Node,
    Ch_Type

FROM dbo.Prod_PM_all
WHERE 
    -- Filter to last 3 years
    Reset_Date >= DATEADD(YEAR, -3, GETDATE())
    -- Exclude NULL part counts
    AND Part_Count IS NOT NULL
    -- Exclude NULL FAB_ENTITY
    AND FAB_ENTITY IS NOT NULL
    -- Exclude NULL Part names
    AND Part IS NOT NULL;

GO

-- =============================================
-- Verify view creation
-- =============================================
IF OBJECT_ID('dbo.vw_Part_Performance_3yr', 'V') IS NOT NULL
BEGIN
    PRINT '';
    PRINT '✓ View dbo.vw_Part_Performance_3yr created successfully!';
    PRINT '';
    
    -- Get row count
    DECLARE @RowCount INT;
    SELECT @RowCount = COUNT(*) FROM dbo.vw_Part_Performance_3yr;
    PRINT 'Total part replacement events: ' + CAST(@RowCount AS VARCHAR(20));
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
-- Sample data: Show parts with frequent replacements
-- =============================================
PRINT 'Sample Data: Parts with Frequent Replacements (Top 20)';
PRINT '------------------------------------------------------------';

SELECT TOP 20
    Part,
    FAB_ENTITY,
    CEID,
    replacement_date,
    part_life_wafers,
    days_since_last_replacement,
    replacement_frequency_category,
    part_life_category
FROM dbo.vw_Part_Performance_3yr
WHERE days_since_last_replacement IS NOT NULL  -- Exclude first installations
ORDER BY days_since_last_replacement ASC;  -- Shortest replacement intervals first

GO

-- =============================================
-- Sample data: Show parts with longest life
-- =============================================
PRINT '';
PRINT 'Sample Data: Parts with Longest Life (Top 20)';
PRINT '------------------------------------------------------------';

SELECT TOP 20
    Part,
    FAB_ENTITY,
    CEID,
    replacement_date,
    part_life_wafers,
    days_since_last_replacement,
    replacement_frequency_category,
    part_life_category
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers IS NOT NULL
ORDER BY part_life_wafers DESC;  -- Longest lasting parts first

GO

-- =============================================
-- Summary statistics by part type
-- =============================================
PRINT '';
PRINT 'Summary Statistics by Part Type (Top 20 Parts):';
PRINT '------------------------------------------------------------';

SELECT TOP 20
    Part,
    COUNT(*) AS total_replacements,
    AVG(part_life_wafers) AS avg_part_life_wafers,
    MIN(part_life_wafers) AS min_part_life_wafers,
    MAX(part_life_wafers) AS max_part_life_wafers,
    AVG(CAST(days_since_last_replacement AS FLOAT)) AS avg_days_between_replacements,
    COUNT(DISTINCT FAB_ENTITY) AS tools_with_this_part
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers > 0  -- Exclude zero/NULL
GROUP BY Part
ORDER BY COUNT(*) DESC;  -- Most frequently replaced parts first

GO

-- =============================================
-- Data quality checks
-- =============================================
PRINT '';
PRINT 'Data Quality Checks:';
PRINT '------------------------------------------------------------';

SELECT 
    'Total part replacement events' AS check_name,
    COUNT(*) AS count_value
FROM dbo.vw_Part_Performance_3yr

UNION ALL

SELECT 
    'Unique parts tracked',
    COUNT(DISTINCT Part)
FROM dbo.vw_Part_Performance_3yr

UNION ALL

SELECT 
    'Unique tools tracked',
    COUNT(DISTINCT FAB_ENTITY)
FROM dbo.vw_Part_Performance_3yr

UNION ALL

SELECT 
    'Replacements with previous history',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE previous_replacement_date IS NOT NULL

UNION ALL

SELECT 
    'First-time installations (no history)',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE previous_replacement_date IS NULL

UNION ALL

SELECT 
    'Parts with very short life (<5K wafers)',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers < 5000

UNION ALL

SELECT 
    'Parts with very long life (>50K wafers)',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers > 50000;

GO

-- =============================================
-- Validation: Check for anomalies
-- =============================================
PRINT '';
PRINT 'Anomaly Detection:';
PRINT '------------------------------------------------------------';

-- Parts replaced too frequently (< 1 week)
SELECT 
    'Parts replaced < 1 week apart' AS anomaly_type,
    COUNT(*) AS anomaly_count
FROM dbo.vw_Part_Performance_3yr
WHERE days_since_last_replacement < 7
  AND days_since_last_replacement IS NOT NULL

UNION ALL

-- Parts with suspiciously low wafer counts
SELECT 
    'Parts replaced with <100 wafers',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers < 100

UNION ALL

-- Parts with suspiciously high wafer counts
SELECT 
    'Parts with >100K wafers (possible counter issue)',
    COUNT(*)
FROM dbo.vw_Part_Performance_3yr
WHERE part_life_wafers > 100000;

GO

PRINT '';
PRINT '====================================================================';
PRINT 'View Creation Complete!';
PRINT '====================================================================';
PRINT '';
PRINT 'PURPOSE:';
PRINT '  - Track individual part performance and reliability';
PRINT '  - Show how long parts last before replacement';
PRINT '  - Identify parts that fail frequently vs rarely';
PRINT '  - Support part investigation analysis in Power BI';
PRINT '';
PRINT 'USE CASES:';
PRINT '  - Page 5: PM Life & Part Investigation';
PRINT '  - Part cost analysis (identify high-replacement parts)';
PRINT '  - Part reliability trends';
PRINT '  - Process node comparison (do parts last longer in certain processes?)';
PRINT '';
PRINT 'KEY COLUMNS:';
PRINT '  - part_life_wafers: How many wafers this part handled before replacement';
PRINT '  - days_since_last_replacement: How long part was installed';
PRINT '  - replacement_frequency_category: Grouped frequency (frequent/normal/rare)';
PRINT '  - part_life_category: Grouped part life (short/average/long)';
PRINT '';
PRINT 'Next Steps:';
PRINT '  1. Review sample data to validate part life calculations';
PRINT '  2. Check summary statistics for each part type';
PRINT '  3. Investigate any anomalies flagged';
PRINT '  4. Ready to import into Power BI for part analysis pages';
PRINT '';
PRINT 'View Name: dbo.vw_Part_Performance_3yr';
PRINT 'Database: Parts_Counter_Production';
PRINT '';
GO
