// 2) Category items, normalized Category → text via IfError (works for Choice or Text)
ClearCollect(
    colModuleCatItems_Working,
    With(
        {
            src: AddColumns(
                    Skill_Matrix_CategoryItems,
                    CategoryText,
                    With({ c: Category }, IfError(c.Value, c))   // <-- FIXED: no Text(), no quotes
                 )
        },
        AddColumns(
            Filter(src, CategoryText = Text(cmbSelectCategory.Selected.Value)),
            IsSelectedForModule,
            CountIf(colRefActiveIds, CatItem_ID = ThisRecord.CatItem_ID) > 0
        )
    )
);
