Set(
    varModuleIdSafe,
    Coalesce(
        // 1) selected module id (text)
        If(
            !IsBlank(varSelectedModuleId) && Len(Trim(Text(varSelectedModuleId))) > 0,
            Text(varSelectedModuleId),
            ""                                  // << force text
        ),
        // 2) lookup by module name (text)
        Text( LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID) ),
        // 3) fallback from AFTER snapshot (use LookUp(true, …) instead of First(...) to avoid empty-table issues)
        Text( LookUp(colRefAfterSave,  true, Mod_ID) ),
        // 4) fallback from BEFORE snapshot
        Text( LookUp(colRefBeforeSave, true, Mod_ID) ),
        ""                                      // final text fallback
    )
);
