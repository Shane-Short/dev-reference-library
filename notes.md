4-Week Rolling PM Count = 
VAR CurrentYearWW = MAX(pm_flex_enriched[YEARWW])
VAR CurrentYear = VALUE(LEFT(CurrentYearWW, 4))
VAR CurrentWeek = VALUE(RIGHT(CurrentYearWW, 2))
RETURN
    CALCULATE(
        [Total PM Events],
        FILTER(
            ALL(pm_flex_enriched[YEARWW], pm_flex_enriched[ww_year], pm_flex_enriched[ww_number]),
            VAR RowYear = pm_flex_enriched[ww_year]
            VAR RowWeek = pm_flex_enriched[ww_number]
            VAR WeekDiff = 
                IF(
                    RowYear = CurrentYear,
                    CurrentWeek - RowWeek,
                    IF(
                        RowYear = CurrentYear - 1,
                        52 + CurrentWeek - RowWeek,
                        999
                    )
                )
            RETURN WeekDiff >= 0 && WeekDiff <= 3
        )
    )





4-Week Rolling Avg PM Life = 
VAR CurrentYearWW = MAX(pm_flex_enriched[YEARWW])
VAR CurrentYear = VALUE(LEFT(CurrentYearWW, 4))
VAR CurrentWeek = VALUE(RIGHT(CurrentYearWW, 2))
RETURN
    CALCULATE(
        [Average PM Life],
        FILTER(
            ALL(pm_flex_enriched[YEARWW], pm_flex_enriched[ww_year], pm_flex_enriched[ww_number]),
            VAR RowYear = pm_flex_enriched[ww_year]
            VAR RowWeek = pm_flex_enriched[ww_number]
            VAR WeekDiff = 
                IF(
                    RowYear = CurrentYear,
                    CurrentWeek - RowWeek,
                    IF(
                        RowYear = CurrentYear - 1,
                        52 + CurrentWeek - RowWeek,
                        999
                    )
                )
            RETURN WeekDiff >= 0 && WeekDiff <= 3
        )
    )





4-Week Rolling Downtime Hours = 
VAR CurrentYearWW = MAX(pm_flex_enriched[YEARWW])
VAR CurrentYear = VALUE(LEFT(CurrentYearWW, 4))
VAR CurrentWeek = VALUE(RIGHT(CurrentYearWW, 2))
RETURN
    CALCULATE(
        [Total Downtime Hours],
        FILTER(
            ALL(pm_flex_enriched[YEARWW], pm_flex_enriched[ww_year], pm_flex_enriched[ww_number]),
            VAR RowYear = pm_flex_enriched[ww_year]
            VAR RowWeek = pm_flex_enriched[ww_number]
            VAR WeekDiff = 
                IF(
                    RowYear = CurrentYear,
                    CurrentWeek - RowWeek,
                    IF(
                        RowYear = CurrentYear - 1,
                        52 + CurrentWeek - RowWeek,
                        999
                    )
                )
            RETURN WeekDiff >= 0 && WeekDiff <= 3
        )
    )




Chronic Tools % = 
DIVIDE(
    [Total Chronic Tools],
    DISTINCTCOUNT(pm_flex_enriched[ENTITY]),
    0
)






Average Chronic Score = AVERAGE(pm_flex_chronic_tools[chronic_score])






Critical Chronic Tools = 
CALCULATE(
    COUNTROWS(pm_flex_chronic_tools),
    pm_flex_chronic_tools[chronic_severity] = "Critical"
)







High Chronic Tools = 
CALCULATE(
    COUNTROWS(pm_flex_chronic_tools),
    pm_flex_chronic_tools[chronic_severity] = "High"
)





Medium Chronic Tools = 
CALCULATE(
    COUNTROWS(pm_flex_chronic_tools),
    pm_flex_chronic_tools[chronic_severity] = "Medium"
)




Low Chronic Tools = 
CALCULATE(
    COUNTROWS(pm_flex_chronic_tools),
    pm_flex_chronic_tools[chronic_severity] = "Low"
)




Average Data Quality Score = AVERAGE(pm_flex_enriched[data_quality_score])
