// 1. Ensure the working collection exists with the shape we expect,
//    but do NOT seed it with a blank row.
If(
    !IsCollection(colModulesInPreset_New),
    ClearCollect(
        colModulesInPreset_New,
        Table(
            // create the table with column names, but no rows:
            { Modules: Blank(), Module_ID: Blank() }
        )
    );
    // then immediately clear it so we don't keep that stub row
    Clear(colModulesInPreset_New)
);

// 2. Pull name + id from the selected module in the combo box.
//    Use Coalesce so we work with either ModuleName/Module_ID
//    or Title/Mod_ID depending on how colAllModules was built.
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
                Modules: pickedName,
                Module_ID: pickedId
            }
        )
    )
);

// 4. Hard cleanup just in case: remove any accidental blank rows
//    (Module_ID = Blank()) that might have slipped in.
If(
    CountRows(
        Filter(
            colModulesInPreset_New,
            IsBlank(Module_ID)
        )
    ) > 0,
    RemoveIf(
        colModulesInPreset_New,
        IsBlank(Module_ID)
    )
);

// 5. Reset the combo so user can pick another
Reset(cmbPresetModules)
