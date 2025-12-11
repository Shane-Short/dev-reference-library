FACILITY = 
VAR OriginalFab = Auto_Tool_List[Fab]
VAR CleanFab = 
    SWITCH(
        TRUE(),
        // Special case: D1X → D1D
        CONTAINSSTRING(UPPER(OriginalFab), "D1X"), "D1D",
        // Extract INT.FAB## format
        CONTAINSSTRING(UPPER(OriginalFab), "FAB"), 
            "F" & TRIM(MID(OriginalFab, SEARCH("FAB", UPPER(OriginalFab)) + 3, 10)),
        // Extract INT## format
        CONTAINSSTRING(UPPER(OriginalFab), "INT"), 
            "F" & TRIM(MID(OriginalFab, SEARCH("INT", UPPER(OriginalFab)) + 3, 10)),
        // Fallback
        OriginalFab
    )
RETURN CleanFab
