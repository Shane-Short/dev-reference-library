If(
    !IsEmpty(cmbPresetModules.SelectedItems),
    With(
        {
            sel: First(cmbPresetModules.SelectedItems),
            thisName: sel.ModuleName,     // or sel.Title if that’s your field
            thisId:   sel.Mod_ID          // or sel.Module_ID — match your schema
        },
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_New,
                    Module_ID = thisId
                )
            ),
            Collect(
                colModulesInPreset_New,
                {
                    Modules: thisName,
                    Module_ID: thisId
                }
            )
        )
    );
    // clear selection safely without Reset()
    Reset(cmbPresetModules)
)
