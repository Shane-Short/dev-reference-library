// 4. Deactivate removed modules in Team_Presets

// 4.1 Rebuild the helper collection in a way that guarantees column names
Clear(colPresetRowsToDeactivate);

ForAll(
    colRemovedModules As goneModule,
    With(
        {
            matchesForThisModule:
                Filter(
                    Skill_Matrix_Team_Presets,
                    Preset_ID = varSelectedPresetId,
                    IsActive = true,
                    (
                        Module_ID = goneModule.Module_ID
                        ||
                        Modules = goneModule.ModuleName
                    )
                )
        },
        // Collect each matching row with stable column names
        ForAll(
            matchesForThisModule As m,
            Collect(
                colPresetRowsToDeactivate,
                {
                    RowID: m.ID,
                    RowModuleID: m.Module_ID,
                    RowModuleName: m.Modules
                }
            )
        )
    )
);

// 4.2 Now patch each row in that helper collection
ForAll(
    colPresetRowsToDeactivate As deadRow,
    Patch(
        Skill_Matrix_Team_Presets,
        LookUp(
            Skill_Matrix_Team_Presets,
            ID = deadRow.RowID
        ),
        {
            IsActive: false,
            Updated_By: User().Email,
            Updated_At: Now()
        }
    )
);

// 4.3 Cleanup helper (optional but nice)
Clear(colPresetRowsToDeactivate);
