// 0) Normalize who we are
Set(varMe, Lower(User().Email));

// 1) Start from all this user’s entries
ClearCollect(
    colUserEntries_All,
    Filter(
        Skill_Matrix_Entries,
        Lower(Employee_Email) = varMe
    )
);

// 2) Apply optional UI filters (preset/module/category) if you have them
//    (if you don't have one of these filters, just remove that line)
ClearCollect(
    colUserEntries,
    Filter(
        colUserEntries_All,
        // preset filter
        If(IsBlank(cmbHomePreset.Selected.Preset_ID), true, Preset_ID = cmbHomePreset.Selected.Preset_ID),
        // module filter
        If(IsBlank(cmbHomeModule.Selected.Title), true, Module = cmbHomeModule.Selected.Title),
        // category filter
        If(IsBlank(cmbHomeCategory.Selected.Value), true, Category = cmbHomeCategory.Selected.Value)
    )
);

// 3) Totals label can bind to CountRows(colUserEntries)


ClearCollect(
    colPie_LevelBreakdown,
    ForAll(
        Sequence(5),    // produces 1..5
        With(
            { lvl: Value },
            {
                Label: "Level " & Text(lvl),
                Count: CountIf(colUserEntries, Value(Skill_Level) = lvl)
            }
        )
    )
);

ClearCollect(
    colPie_CategoryBreakdown,
    With(
        {
            g:
            AddColumns(
                GroupBy(colUserEntries, "Category", "Rows"),
                "Count", CountRows(Rows)
            )
        },
        // take top 10 by volume; fold the rest into "Other" (optional)
        If(
            CountRows(g) <= 10,
            g,
            Collect(
                ClearCollect(col__TopCats,
                    FirstN(SortByColumns(g, "Count", Descending), 10)
                ),
                col__TopCats
            );
            Collect(
                col__TopCats,
                {
                    Category: "Other",
                    Rows:    Table({}),   // just to satisfy schema
                    Count:   Sum(FirstN(SortByColumns(g, "Count", Descending), CountRows(g) - 10), Count)
                }
            );
            col__TopCats
        )
    )
);


ClearCollect(
    colPie_TypeBreakdown,
    With(
        {
            g: AddColumns(
                GroupBy(colUserEntries, "Skill_Type", "Rows"),
                "Count", CountRows(Rows)
            )
        },
        g
    )
);
// Bind to a 3rd pie: Items = colPie_TypeBreakdown (Labels=Skill_Type, Values=Count)
