// 1. Cache which preset we chose
Set(varSelectedPresetId, cmbEditPreset.Selected.Value);
Set(varSelectedPresetTitle, cmbEditPreset.Selected.Title);

// 2. Build a normalized table of active modules for this preset
//    Shape: Module_ID, ModuleName
ClearCollect(
    colModulesInPreset_Original,
    AddColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId,
                IsActive = true
            ),
            "Module_ID",
            "Modules"            // readable module name column in SP
        ),
        "ModuleName", Modules
    )
);

// 3. Working copy (the only one we will edit from here on out)
ClearCollect(
    colModulesInPreset_Edit_Working,
    colModulesInPreset_Original
);

// 4. Reset any local "confirm remove" UI state if you’re using it
UpdateContext({ varConfirmRemove: false });
Set(varModuleToRemove, "");


// remove this row from the WORKING collection only
RemoveIf(
    colModulesInPreset_Edit_Working,
    Module_ID = ThisItem.Module_ID
);



If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            selModuleId: cmbAddModules_Edit.Selected.Mod_ID,      // rename if yours is Module_ID
            selModuleName: cmbAddModules_Edit.Selected.Title      // rename if yours is ModuleName
        },
        // only add if this Module_ID not already in working
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = selModuleId
                )
            ),
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Module_ID: selModuleId,
                    ModuleName: selModuleName
                }
            )
        )
    );
    Reset(cmbAddModules_Edit)
);




ClearCollect(
    colAddedModules,
    Filter(colModulesInPreset_Edit_Working, IsBlank(LookUp(colModulesInPreset_Edit, Module_ID = Module_ID)))
);

ClearCollect(
    colRemovedModules,
    Filter(colModulesInPreset_Edit, IsBlank(LookUp(colModulesInPreset_Edit_Working, Module_ID = Module_ID)))
);




// ADDED = in working now, but not in original snapshot
ClearCollect(
    colAddedModules,
    Filter(
        colModulesInPreset_Edit_Working,
        IsBlank(
            LookUp(
                colModulesInPreset_Original,
                Module_ID = Module_ID
            )
        )
    )
);

// REMOVED = was in original snapshot, but not in working now
ClearCollect(
    colRemovedModules,
    Filter(
        colModulesInPreset_Original,
        IsBlank(
            LookUp(
                colModulesInPreset_Edit_Working,
                Module_ID = Module_ID
            )
        )
    )
);


