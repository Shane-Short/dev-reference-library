// 1. Cache the selected preset's ID and Title into globals
Set(
    varSelectedPresetId,
    cmbEditPreset.Selected.Value      // <-- this is Preset_ID
);
Set(
    varSelectedPresetTitle,
    cmbEditPreset.Selected.Title      // <-- this is the preset name
);

// 2. Load that preset's module rows from the SharePoint list
//    We pull rows for just this Preset_ID, then project them into a clean shape:
//    { ModuleName: <Modules column>, Module_ID: <Module_ID column> }

ClearCollect(
    colModulesInPreset_Edit,
    AddColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId   // use the var we just set
            ),
            "Modules",
            "Module_ID"
        ),
        "ModuleName", Modules
    )
);

// 3. Make the working copy we'll mutate while editing
ClearCollect(
    colModulesInPreset_Edit_Working,
    colModulesInPreset_Edit
);

// 4. Reset downstream remove-confirm UI (you'll hook this up later if you haven't already)
UpdateContext({ varConfirmRemove: false });
Set(varModuleToRemove, "");
