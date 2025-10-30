If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            selId: cmbAddModules_Edit.Selected.Module_ID,
            selName: cmbAddModules_Edit.Selected.ModuleName,

            existingRow:
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = selId
                )
        },
        If(
            // already exists in working list
            !IsBlank(existingRow),
            Patch(
                colModulesInPreset_Edit_Working,
                existingRow,
                { IsActive: true }
            ),
            // otherwise add as brand new row
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Module_ID: selId,
                    ModuleName: selName,
                    IsActive: true
                }
            )
        )
    )
);

// IMPORTANT: clear selection so ComboBox is "free" again
Set(varModulePick_Edit, Blank())
