If(
    !IsBlank(cmbPresetModules.Selected.Title),

    With(
        {
            thisName: cmbPresetModules.Selected.Title,
            thisId: cmbPresetModules.Selected.Mod_ID    // <-- swap to .Module_ID if that's the real column
        },

        // only add if this Module_ID is not already present in the collection
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_New,
                    Module_ID = thisId    // compare by ID, not just name
                )
            ),
            Collect(
                colModulesInPreset_New,
                {
                    Modules: thisName,    // human-readable (what the preset UI shows)
                    Module_ID: thisId     // machine-readable, what Flow needs later
                }
            )
        )
    )
);
