With(
    {
        // All modules in the system shaped the same way
        allMods: AddColumns(
            colAllModules,
            "ModuleName", Title,
            "Module_ID", Mod_ID
        ),
        // Only modules still "live" in the working list
        activeMods: Filter(
            colModulesInPreset_Edit_Working,
            IsActive = true
        )
    },
    // We only show modules whose Module_ID is NOT already in activeMods
    Filter(
        allMods,
        IsBlank(
            LookUp(
                activeMods,
                Module_ID = Module_ID
            )
        )
    )
)
