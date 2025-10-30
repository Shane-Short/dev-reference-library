// 4. Soft-remove (IsActive=false) for anything in colRemovedModules

// 4.1 Build a list of SharePoint IDs we want to deactivate
ClearCollect(
    colPresetRowsToDeactivate,
    ForAll(
        colRemovedModules As gone,
        ForAll(
            Filter(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId,
                IsActive = true,
                (
                    Module_ID = gone.Module_ID
                    ||
                    Modules = gone.ModuleName
                )
            ) As matchRow,
            {
                RowID: matchRow.ID,          // <- we CONTROL the name now
                RowModuleID: matchRow.Module_ID,
                RowModuleName: matchRow.Modules
            }
        )
    )
);

// 4.2 Loop those RowIDs, Patch each one by LookUp()
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

// 4.3 Cleanup helper
Clear(colPresetRowsToDeactivate);
