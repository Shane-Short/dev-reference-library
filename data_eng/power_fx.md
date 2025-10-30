SortByColumns(
    Filter(
        colAllModules,
        With(
            {
                candidateName: ModuleName,
                candidateId:   Mod_ID
            },
            CountIf(
                colModulesInPreset_Edit_Working,
                Module_ID = candidateId ||
                ModuleName = candidateName
            ) = 0
        )
    ),
    "ModuleName",
    SortOrder.Ascending
)


With(
    {
        selName: cmbAddModules_Edit.Selected.ModuleName,
        selId:   cmbAddModules_Edit.Selected.Mod_ID
    },
    If(
        !IsBlank(selId),
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
                    ModuleName: selName,
                    Module_ID:  selId
                }
            )
        )
    );
    // Reset the picker so you can immediately pick another one without the dropdown dying
    Reset(cmbAddModules_Edit)
)
