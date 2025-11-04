// cache selected category text
Set(varSelectedCategoryText, Text(cmbSelectCategory.Selected.Value));

// 1) Build a clean ref-id list for THIS module only (one column: id as text)
ClearCollect(
    colRefActiveIds,
    AddColumns(
        Distinct(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && IsActive = true
            ),
            Text(CatItem_ID)   // Distinct over the text value
        ),
        id, Text(Coalesce(Result, Value)) // handle either 'Result' or 'Value'
    )
);

// 2) Build the category working set with a stable id and correct check state
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
            AddColumns(src, id, Text(CatItem_ID)),
            IsSelectedForModule,
                !IsBlank(
                    LookUp(
                        colRefActiveIds,
                        id = ThisRecord.id
                    )
                )
        )
    )
);





