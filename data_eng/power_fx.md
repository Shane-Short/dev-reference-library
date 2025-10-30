If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            selId:
                Coalesce(
                    cmbAddModules_Edit.Selected.Module_ID_Norm,
                    cmbAddModules_Edit.Selected.Module_ID,
                    ""
                ),
            selName:
                Coalesce(
                    cmbAddModules_Edit.Selected.ModuleName_Norm,
                    cmbAddModules_Edit.Selected.ModuleName,
                    ""
                )
        },

        // If the module already exists in our working set, just reactivate it
        If(
            !IsBlank(
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = selId ||
                    ModuleName = selName
                )
            ),

            UpdateIf(
                colModulesInPreset_Edit_Working,
                Module_ID = selId ||
                ModuleName = selName,
                {
                    IsActive: true
                }
            ),

            // else it's brand new to this preset working list
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Module_ID:  selId,
                    ModuleName: selName,
                    IsActive:   true
                }
            )
        )
    );

    Reset(cmbAddModules_Edit)
)
