If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            selName: cmbAddModules_Edit.Selected.ModuleName,
            selId:   cmbAddModules_Edit.Selected.Module_ID
        },
        // only collect if we don't already have this module ID in the working list
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = selId
                )
            ),
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Modules: selName,    // human-readable to show in gallery
                    Module_ID: selId     // actual ID to persist to SharePoint
                }
            )
        )
    );
    // reset after each selection so dropdown doesn't lock up
    Reset(cmbAddModules_Edit)
);
