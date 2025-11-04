ClearCollect(
    colRefActiveIds,
    With(
        {
            src:
                ShowColumns(
                    Filter(
                        Skill_Matrix_Reference,
                        Mod_ID = varSelectedModuleId && IsActive = true
                    ),
                    "CatItem_ID"
                )
        },
        // Distinct() coerces to a single 'Result' column and removes dupes
        AddColumns(
            Distinct(src, Text(CatItem_ID)),
            id, Result
        )
    )
);



IsSelectedForModule,
If(
    IsBlank(id),
    false,
    CountIf(colRefActiveIds, id = id) > 0
)



