// 2. Build a normalized table of active modules for this preset
//    Shape: Module_ID, ModuleName
ClearCollect(
    colModulesInPreset_Original,
    ForAll(
        Filter(
            Skill_Matrix_Team_Presets,
            Preset_ID = varSelectedPresetId,
            IsActive = true
        ) As row,
        With(
            {
                resolvedId:
                    If(
                        !IsBlank(row.Module_ID),
                        row.Module_ID,
                        LookUp(
                            Skill_Matrix_Modules,
                            Title = row.Modules   /* human name match */,
                            Mod_ID                /* <-- actual ID column in Modules list */
                        )
                    ),
                resolvedName:
                    If(
                        !IsBlank(row.Modules),
                        row.Modules,
                        LookUp(
                            Skill_Matrix_Modules,
                            Mod_ID = row.Module_ID,
                            Title
                        )
                    )
            },
            {
                Module_ID: resolvedId,
                ModuleName: resolvedName
            }
        )
    )
);



