// 4. Soft-remove (IsActive=false) for anything in colRemovedModules

// 4.1 Build a lightweight table of row IDs we want to deactivate
ClearCollect(
    colPresetRowsToDeactivate,
    ForAll(
        colRemovedModules As gone,
        ShowColumns(
            Filter(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId,
                IsActive = true,
                (
                    Module_ID = gone.Module_ID
                    ||
                    Modules = gone.ModuleName
                )
            ),
            "ID"    // <-- ONLY keep the SharePoint ID so it's patchable
        )
    )
);

// 4.2 Loop those IDs, Patch each one by LookUp()
ForAll(
    colPresetRowsToDeactivate As deadRow,
    Patch(
        Skill_Matrix_Team_Presets,
        LookUp(
            Skill_Matrix_Team_Presets,
            ID = deadRow.ID
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
