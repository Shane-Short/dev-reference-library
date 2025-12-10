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
