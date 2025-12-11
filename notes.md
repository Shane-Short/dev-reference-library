FACILITY = 
VAR OriginalFab = UPPER(Auto_Tool_List[Fab])
VAR CleanFab = 
    SWITCH(
        TRUE(),
        // D1D group (includes D1C, D1D, D1X, AFO, RP1)
        OR(
            CONTAINSSTRING(OriginalFab, "D1C"),
            CONTAINSSTRING(OriginalFab, "D1D"),
            CONTAINSSTRING(OriginalFab, "D1X"),
            CONTAINSSTRING(OriginalFab, "AFO"),
            CONTAINSSTRING(OriginalFab, "RP1")
        ), "D1D",
        
        // F24 group (24, 34)
        OR(
            CONTAINSSTRING(OriginalFab, "24"),
            CONTAINSSTRING(OriginalFab, "34")
        ), "F24",
        
        // F28 group
        CONTAINSSTRING(OriginalFab, "28"), "F28",
        
        // F32 group (12, 32, 42, 52)
        OR(
            CONTAINSSTRING(OriginalFab, "12"),
            CONTAINSSTRING(OriginalFab, "32"),
            CONTAINSSTRING(OriginalFab, "42"),
            CONTAINSSTRING(OriginalFab, "52")
        ), "F32",
        
        // F11 group
        CONTAINSSTRING(OriginalFab, "11"), "F11",
        
        // Malaysia group
        CONTAINSSTRING(OriginalFab, "MAL"), "MAL",
        
        // Fallback - keep original
        OriginalFab
    )
RETURN CleanFab
