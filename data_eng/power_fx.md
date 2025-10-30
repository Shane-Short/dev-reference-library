If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            pickedId:   cmbAddModules_Edit.Selected._ModuleID,
            pickedName: cmbAddModules_Edit.Selected._ModuleName
        },
        // See if it's already in the working collection
        With(
            {
                existingRow: LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = pickedId
                )
            },
            If(
                // If we already had it (maybe it was IsActive=false), revive it
                !IsBlank(existingRow),
                Patch(
                    colModulesInPreset_Edit_Working,
                    existingRow,
                    {
                        Module_ID: pickedId,
                        ModuleName: pickedName,
                        IsActive: true
                    }
                ),
                // Otherwise, add it fresh
                Collect(
                    colModulesInPreset_Edit_Working,
                    {
                        Module_ID: pickedId,
                        ModuleName: pickedName,
                        IsActive: true
                    }
                )
            )
        )
    )
);

// Hard-clear ComboBox selection to avoid the chevron glitch
Set(varModulePick_Edit, "");
