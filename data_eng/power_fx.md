AddColumns(
    SortByColumns(
        Filter(
            colAllModules,
            With(
                { curName: Title },
                CountIf(colModulesInPreset_New, Modules = curName) = 0
            )
        ),
        "Title",
        SortOrder.Ascending
    ),
    "Mod_ID", Mod_ID
)
