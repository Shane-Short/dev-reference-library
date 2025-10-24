// source = what’s currently in view for the user
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
    // Build a small summary table: one row per distinct Skill_Level
    AddColumns(
        // make distinct Skill_Level values
        RenameColumns(
            Distinct(src, Skill_Level),
            "Result", "Level"
        ),
        // Count how many rows at each level
        "Value", CountIf(src, Skill_Level = Level),
        // Text to show in legend/slices
        "Category", "Level " & Text(Coalesce(Level, 1))
    )
)
