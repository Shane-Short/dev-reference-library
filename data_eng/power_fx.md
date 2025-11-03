With(
    { catText: varSelectedCategoryText },

    // base set with guaranteed schema (even when empty)
    With(
        {
            baseRows:
                If(
                    IsBlank(varSelectedModuleId) || IsBlank(catText),
                    // return 0 rows, but from the real table so schema exists
                    FirstN(
                        Filter(Skill_Matrix_CategoryItems, false),
                        0
                    ),
                    Filter(Skill_Matrix_CategoryItems, Text(Category) = catText)
                )
        },
        // add normalized id + selected flag in BOTH branches
        AddColumns(
            AddColumns(
                baseRows,
                id, Text(CatItem_ID)
            ),
            IsSelectedForModule,
            CountIf(
                colRefActiveIds,
                id = ThisRecord.id
            ) > 0
        )
    )
)
