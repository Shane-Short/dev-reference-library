With(
    {
        // 1. Normalize what's currently in this preset (what's in the gallery)
        curActiveModsNormalized:
            AddColumns(
                colModulesInPreset_Edit_Working,
                // Force consistent column names, even if they didn't exist before
                "Module_ID_Norm",
                    Coalesce(
                        Module_ID,
                        Mod_ID,    // fallback if it's still called Mod_ID in some rows
                        ""         // last resort
                    ),
                "ModuleName_Norm",
                    Coalesce(
                        ModuleName,
                        Title,     // fallback if old rows still had Title instead of ModuleName
                        Modules,   // fallback if old rows still had Modules instead of Title
                        ""
                    ),
                "ActiveFlag",
                    Coalesce(
                        IsActive,
                        true       // if IsActive column didn't exist in older rows, treat as true
                    )
            ),

        // 2. Normalize the master module list (everything you COULD add)
        allModsNormalized:
            AddColumns(
                colAllModules,
                "Module_ID_Norm",
                    Coalesce(
                        Mod_ID,
                        Module_ID,
                        ""        // just in case
                    ),
                "ModuleName_Norm",
                    Coalesce(
                        ModuleName,
                        Title,
                        ""
                    )
            )
    },

    // 3. Now build the dropdown list:
    SortByColumns(
        Filter(
            allModsNormalized,
            // Keep this module ONLY if it's not already Active in the working preset
            CountIf(
                curActiveModsNormalized,
                ActiveFlag = true &&
                (
                    // Prefer ID match when both sides have IDs
                    (!IsBlank(Module_ID_Norm) && Module_ID_Norm = ThisRecord.Module_ID_Norm)
                    ||
                    // Fallback: match by name if one/both rows didn't have IDs
                    (IsBlank(Module_ID_Norm) && ModuleName_Norm = ThisRecord.ModuleName_Norm)
                )
            ) = 0
        ),
        "ModuleName_Norm",
        SortOrder.Ascending
    )
)
