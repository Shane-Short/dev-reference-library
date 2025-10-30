With(
    {
        allMods:
            AddColumns(
                colAllModules,
                "_ModuleName", Title,
                "_ModuleID", Mod_ID
            ),
        activeMods:
            Filter(
                colModulesInPreset_Edit_Working,
                IsActive = true
            )
    },
    // return all modules that are NOT already active in the preset
    Filter(
        allMods,
        IsBlank(
            LookUp(
                activeMods,
                Module_ID = _ModuleID
            )
        )
    )
)
