Set(
    varModuleIdSafe,
    Coalesce(
        // 1) selected module id if present (coerced to text)
        If(
            !IsBlank(varSelectedModuleId) && Len(Trim(Text(varSelectedModuleId))) > 0,
            Text(varSelectedModuleId)
        ),
        // 2) lookup by module name (coerced to text)
        Text(LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID)),
        // 3) fallback from AFTER snapshot
        Text(First(colRefAfterSave).Mod_ID),
        // 4) fallback from BEFORE snapshot
        Text(First(colRefBeforeSave).Mod_ID)
    )
);
