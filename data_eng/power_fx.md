"Skill Level Distribution - " &
If(
    !IsBlank(drpModule.Selected.Title) && !IsBlank(drpCategory.Selected.Result),
    drpModule.Selected.Title & " / " & drpCategory.Selected.Result,
    If(
        !IsBlank(drpModule.Selected.Title),
        drpModule.Selected.Title,
        "All Modules"
    )
)
