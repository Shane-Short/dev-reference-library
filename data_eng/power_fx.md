// Force varModuleIdSafe to be TEXT first
Set(varModuleIdSafe, "");

// 1) Try the selected module id (coerced to text)
If(
    !IsBlank(varSelectedModuleId) && Len(Text(varSelectedModuleId)) > 0,
    Set(varModuleIdSafe, "" & varSelectedModuleId)
);

// 2) Fallback: lookup by module name (text)
If(
    IsBlank(varModuleIdSafe) && !IsBlank(varSelectedModuleName),
    Set(
        varModuleIdSafe,
        "" & LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID)
    )
);

// 3) Fallback: AFTER snapshot (if present)
If(
    IsBlank(varModuleIdSafe) && CountRows(colRefAfterSave) > 0,
    Set(varModuleIdSafe, "" & First(colRefAfterSave).Mod_ID)
);

// 4) Fallback: BEFORE snapshot (if present)
If(
    IsBlank(varModuleIdSafe) && CountRows(colRefBeforeSave) > 0,
    Set(varModuleIdSafe, "" & First(colRefBeforeSave).Mod_ID)
);
