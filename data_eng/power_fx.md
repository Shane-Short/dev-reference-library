Module_ID:
If(
    CountRows(colRefAfterSave) > 0, "" & First(colRefAfterSave).Mod_ID,
    CountRows(colRefBeforeSave) > 0, "" & First(colRefBeforeSave).Mod_ID,
    !IsBlank(varSelectedModuleId),   "" & varSelectedModuleId,
    ""  // final fallback
),
