// 1. Just in case from previous runs, clear any rows
//    where Module_ID is blank before we add a new one.
If(
    !IsEmpty(
        Filter(
            colModulesInPreset_New,
            IsBlank(Module_ID)
        )
    ),
    RemoveIf(
        colModulesInPreset_New,
        IsBlank(Module_ID)
    )
);

// 2. Grab the selected module's name + ID from the combo box.
//    We Coalesce because sometimes your data is shaped
//    {ModuleName, Module_ID} and sometimes {Title, Mod_ID}.
With(
    {
        pickedName: Coalesce(
            cmbPresetModules.Selected.ModuleName,
            cmbPresetModules.Selected.Title
        ),
        pickedId: Coalesce(
            cmbPresetModules.Selected.Module_ID,
            cmbPresetModules.Selected.Mod_ID
        )
    },

    // 3. Only collect if:
    //    - we actually got an ID
    //    - it's not already in the list
    If(
        !IsBlank(pickedId) &&
        IsBlank(
            LookUp(
                colModulesInPreset_New,
                Module_ID = pickedId
            )
        ),
        Collect(
            colModulesInPreset_New,
            {
                Modules: pickedName,   // human-readable name
                Module_ID: pickedId    // GUID/id we need in Flow
            )
        )
    )
);

// 4. Safety cleanup again (in case Collect injected blanks somehow)
If(
    !IsEmpty(
        Filter(
            colModulesInPreset_New,
            IsBlank(Module_ID)
        )
    ),
    RemoveIf(
        colModulesInPreset_New,
        IsBlank(Module_ID)
    )
);

// 5. Reset the combo so user can pick another module
Reset(cmbPresetModules)
