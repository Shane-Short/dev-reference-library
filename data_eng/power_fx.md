Set(
    varModuleIdSafe,
    Coalesce(
        // 1) selected module id if present and not empty
        If(!IsBlank(varSelectedModuleId) && Trim(varSelectedModuleId) <> "", varSelectedModuleId),
        // 2) lookup by module name
        Text(LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID)),
        // 3) fallback from AFTER snapshot
        Text(First(colRefAfterSave).Mod_ID),
        // 4) fallback from BEFORE snapshot
        Text(First(colRefBeforeSave).Mod_ID)
    )
);
