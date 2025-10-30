With(
    {
        // only actively kept modules in the current preset
        curActiveMods:
            Filter(
                colModulesInPreset_Edit_Working,
                IsActive = true
            )
    },
    SortByColumns(
        Filter(
            colAllModules,                      // full module list from Modules table
            // keep this module if it's NOT already active in the preset
            CountIf(
                curActiveMods,
                // prefer ID match if both sides have an ID
                (!IsBlank(Module_ID) && Module_ID = ThisRecord.Mod_ID)
                ||
                // fallback name match if either side was legacy-without-ID
                (IsBlank(Module_ID) && ModuleName = ThisRecord.Title)
            ) = 0
        ),
        "Title",
        SortOrder.Ascending
    )
)
