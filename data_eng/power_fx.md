Module_ID:
If(
    // 1) use the selected ID if present
    !IsBlank(varSelectedModuleId) && Len(Text(varSelectedModuleId)) > 0,
    "" & varSelectedModuleId,

    // 2) else try Modules list by name
    If(
        !IsBlank(varSelectedModuleName),
        "" & LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID),

        // 3) else fallback to AFTER snapshot
        If(
            CountRows(colRefAfterSave) > 0,
            "" & First(colRefAfterSave).Mod_ID,

            // 4) else fallback to BEFORE snapshot (last resort)
            If(
                CountRows(colRefBeforeSave) > 0,
                "" & First(colRefBeforeSave).Mod_ID,
                ""    // empty string as final text fallback
            )
        )
    )
),
