// Build the source based on current filters (Module/Category)
With(
    {
        src:
            If(
                !IsBlank(drpModule.Selected.Title) && !IsBlank(drpCategory.Selected.Result),
                Filter(colUserSelections, Module = drpModule.Selected.Title && Category = drpCategory.Selected.Result),
                If(
                    !IsBlank(drpModule.Selected.Title),
                    Filter(colUserSelections, Module = drpModule.Selected.Title),
                    colUserSelections
                )
            )
    },
    // Always produce levels 1..5, then count matches in src
    AddColumns(
        ForAll(Sequence(5), { Level: Value }),
        "Qty", CountIf(src, Value(Coalesce(Skill_Level, 1)) = Level),
        "Label", Text(Level)     // slice label should be just "1", "2", ...
    )
)
