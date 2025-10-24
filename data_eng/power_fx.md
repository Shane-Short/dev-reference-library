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
    AddColumns(
        ForAll(Sequence(5), { Level: Value }),
        "Qty", CountIf(src, Value(Coalesce(Skill_Level, 1)) = Level),
        // 👇 Label shows both: the level number and the total
        "Label", Text(Level) & " (" & Text(Qty) & ")"
    )
)
