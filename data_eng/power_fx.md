// 1. Grab the preset the user picked
Set(
    varEditPresetId,
    cmbEditPreset.Selected.Preset_ID
);

// 2. Build the working collection from what’s already in SharePoint
ClearCollect(
    colModulesInPreset_Edit_Working,
    ForAll(
        Filter(
            Skill_Matrix_Team_Presets,
            Preset_ID = varEditPresetId && IsActive = true
        ),
        {
            Modules: Modules,          // readable name in Team_Presets
            Module_ID: Module_ID       // GUID/text we added to Team_Presets
        }
    )
);

// (optional but smart) give the combo box a nudge so it updates Items
UpdateContext({ varEditPresetNudge: Rand() })
