// Snapshot ACTIVE refs for this module with a normalized 'id' column
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            "CatItem_ID"
        ),
        id, Text(CatItem_ID)
    )
);




With(
    {
        src: ShowColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = varSelectedCategoryText
                ),
                "CatItem_ID", "Category", "Item", "Skill_Type"
            )
    },
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            AddColumns(src, id, Text(CatItem_ID)),
            IsSelectedForModule,
                With(
                    { tgl: LookUp(colToggleLog, CatItem_ID = CatItem_ID) },
                    If(
                        IsBlank(tgl),
                        CountIf(colRefActiveIds, id = ThisRecord.id) > 0,
                        tgl.Desired
                    )
                )
        )
    )
);


"active=" & CountRows(colRefActiveIds) & "  first=" & If(CountRows(colModuleCatItems_Working)>0, First(colModuleCatItems_Working).id & ":" & Text(First(colModuleCatItems_Working).IsSelectedForModule), "—")


