Module_ID:
If(
    // 1) Prefer the AFTER snapshot (post-save authoritative)
    CountRows(colRefAfterSave) > 0,
    "" & First(colRefAfterSave).Mod_ID,

    // 2) Else use the BEFORE snapshot
    If(
        CountRows(colRefBeforeSave) > 0,
        "" & First(colRefBeforeSave).Mod_ID,

        // 3) Else use the selected module id if present
        If(
            !IsBlank(varSelectedModuleId),
            "" & varSelectedModuleId,

            // 4) LAST resort: look up by name in Modules
            "" & Text( LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID) )
        )
    )
),
