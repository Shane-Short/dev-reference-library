-- This should return rows (confirms join will work)
SELECT 
    pf.FACILITY_ENTITY,  -- Will create this column in next step
    ws.FAB_ENTITY,
    ws.wafers_produced,
    pf.total_pm_events
FROM pm_flex_enriched pf
INNER JOIN vw_Wafer_Summary_3yr ws
    ON pf.FACILITY + '_' + REPLACE(pf.ENTITY, '_PC', '_PM') = ws.FAB_ENTITY  -- Temporary join logic
    AND pf.YEARWW = ws.WW
WHERE pf.YEARWW >= '2024WW01'  -- Recent data
LIMIT 10;



-- Compare to raw Prod_PM_all to spot-check accuracy
SELECT 
    ws.FAB_ENTITY,
    ws.WW,
    ws.wafers_produced,
    ws.parts_replaced_count,
    -- Compare to raw max part count
    (SELECT MAX(Part_Count) 
     FROM Prod_PM_all 
     WHERE FAB_ENTITY = ws.FAB_ENTITY 
       AND WW = ws.WW) AS raw_max_count
FROM vw_Wafer_Summary_3yr ws
WHERE ws.WW = '2025WW48'  -- Pick a recent week
ORDER BY ws.wafers_produced DESC;



