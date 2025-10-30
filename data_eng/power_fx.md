// ensure the working collection exists in the right shape
If(
    IsEmpty(colModulesInPreset_New),
    ClearCollect(
        colModulesInPreset_New,
        []
    )
);

// Pull name + id from the selected row.
// We try ModuleName / Module_ID first, and fall back to Title / Mod_ID.
With(
    {
        pickedName:
            Coalesce(
                cmbPresetModules.Selected.ModuleName,
                cmbPresetModules.Selected.Title
            ),

        pickedId:
            Coalesce(
                cmbPresetModules.Selected.Module_ID,
                cmbPresetModules.Selected.Mod_ID
            )
    },

    // only add if not already present (compare by pickedId)
    If(
        IsBlank(
            LookUp(
                colModulesInPreset_New,
                Module_ID = pickedId
            )
        ),
        Collect(
            colModulesInPreset_New,
            {
                Modules:   pickedName,
                Module_ID: pickedId
            }
        )
    )
);

// let them pick another module
Reset(cmbPresetModules)
