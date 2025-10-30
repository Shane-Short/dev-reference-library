With(
    {
        // Normalize the current working preset list
        curActiveModsNormalized:
            AddColumns(
                colModulesInPreset_Edit_Working,
                "Module_ID_Norm",
                    Coalesce(
                        Module_ID,
                        ""    // fallback if blank
                    ),
                "ModuleName_Norm",
                    Coalesce(
                        ModuleName,
                        ""    // fallback if blank
                    ),
                "ActiveFlag",
                    Coalesce(
                        IsActive,
                        true  // if IsActive isn't in the collection yet, assume true
                    )
            ),

        // Normalize the master module list (all selectable modules)
        allModsNormalized:
            AddColumns(
                colAllModules,
                "Module_ID_Norm",
                    Coalesce(
                        Module_ID,
                        ""   // should exist, but just in case
                    ),
                "ModuleName_Norm",
                    Coalesce(
                        ModuleName,
                        ""   // should exist
                    )
            )
    },

    // Build the dropdown list = all modules NOT currently active in this preset
    SortByColumns(
        Filter(
            allModsNormalized As cand,
            CountIf(
                curActiveModsNormalized As cur,
                cur.ActiveFlag = true &&
                (
                    // match by ID if both have an ID
                    (
                        !IsBlank(cur.Module_ID_Norm) &&
                        cur.Module_ID_Norm = cand.Module_ID_Norm
                    )
                    ||
                    // OR, if ID is blank just fall back to comparing names
                    (
                        IsBlank(cur.Module_ID_Norm) &&
                        cur.ModuleName_Norm = cand.ModuleName_Norm
                    )
                )
            ) = 0
        ),
        "ModuleName_Norm",
        SortOrder.Ascending
    )
)
