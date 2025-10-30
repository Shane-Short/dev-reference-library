With(
    {
        currentIDs: AddColumns(
            colModulesInPreset_Edit_Working,
            keepId,
            Module_ID
        )
    },
    SortByColumns(
        Filter(
            colAllModules,
            // keep only modules not already in the working preset
            IsBlank(
                LookUp(
                    currentIDs,
                    keepId = Mod_ID
                )
            )
        ),
        "Title",
        SortOrder.Ascending
    )
)



If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            mId: cmbAddModules_Edit.Selected.Mod_ID,
            mName: cmbAddModules_Edit.Selected.Title
        },
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = mId
                )
            ),
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Module_ID: mId,
                    ModuleName: mName
                }
            )
        )
    );
    Reset(cmbAddModules_Edit)
);
