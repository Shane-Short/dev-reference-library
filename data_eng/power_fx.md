With(
    {
        src:
            If(
                IsBlank(drpModule.Selected.Title),
                colUserSelections,                           // all items across all modules
                Filter(colUserSelections, Module = drpModule.Selected.Title)
            )
    },
    SortByColumns(
        AddColumns( Distinct(src, Category), "Value", ThisRecord.Result ),
        "Value",
        Ascending
    )
)
