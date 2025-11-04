// cache selected category text
Set(varSelectedCategoryText, Text(cmbSelectCategory.Selected.Value));

// 1) ACTIVE ref ids for the selected module (shape: [{ id: "…" }])
ClearCollect(
    colRefActiveIds,
    AddColumns(
        Distinct(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && IsActive = true
            ),
            Text(CatItem_ID)            // distinct over the TEXT of CatItem_ID
        ),
        id, Text(Coalesce(Result, Value)) // Distinct column is Result or Value; normalize to 'id'
    )
);

// 2) Working list for this category with correct check state
With(
    {
        src: ShowColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = varSelectedCategoryText
                ),
                "CatItem_ID","Category","Item","Skill_Type"
            )
    },
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            AddColumns(src, id, Text(CatItem_ID)),           // stable text key on each row
            IsSelectedForModule,
                !IsBlank(
                    LookUp(
                        colRefActiveIds,
                        id = ThisRecord.id                    // << REAL membership test
                    )
                )
        )
    )
);
