// 0. Guard: make sure we have at least one module chosen and a preset name
If(
    IsBlank(varPresetName) || CountRows(colModulesInPreset_New) = 0,
    Notify("Please enter a preset name and select at least one module.", NotificationType.Error),
    
    // else:
    With(
        {
            // If you already set a preset GUID somewhere else, keep it.
            // Otherwise generate one here.
            newPresetId:
                Coalesce(
                    varFinalPresetId,
                    GUID()
                ),

            thisUserEmail: User().Email,
            nowTs: Now()
        },

        // 1. Create one row in Skill_Matrix_Team_Presets per selected module
        //    Include both module name and module ID
        ForAll(
            colModulesInPreset_New As m,
            Patch(
                Skill_Matrix_Team_Presets,
                Defaults(Skill_Matrix_Team_Presets),
                {
                    Preset_ID: newPresetId,
                    Title: varPresetName,

                    // human-readable module name
                    Modules: m.Modules,

                    // machine-readable module id
                    Module_ID: m.Module_ID,

                    IsActive: true,
                    Created_By: thisUserEmail,
                    Created_At: nowTs
                }
            )
        );

        // 2. (Optional but smart) update varFinalPresetId so the app "remembers"
        Set(varFinalPresetId, newPresetId);

        // 3. Clear working collection so UI looks clean
        Clear(colModulesInPreset_New);

        // 4. Let the user know
        Notify(
            "Preset '" & varPresetName & "' saved with Module IDs.",
            NotificationType.Success
        )
    )
);
