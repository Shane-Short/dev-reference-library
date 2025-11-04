// 1) Active refs for this module, normalized to single column 'id' (text)
ClearCollect(
    colRefActiveIds,
    RenameColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            "CatItem_ID"
        ),
        "CatItem_ID", "id"
    )
);

// 2) Build the working set for the selected category with precheck flag + carry CatItem_ID
ClearCollect(
    colModuleCatItems_Working,
    AddColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_CategoryItems,
                Text(Category) = varSelectedCategoryText    // ensure you set this in the category picker
            ),
            "CatItem_ID", "Category", "Item", "Skill_Type" // carry CatItem_ID so the checkbox can use it
        ),
        id, Text(CatItem_ID),
        IsSelectedForModule,
        CountIf(colRefActiveIds, id = Text(CatItem_ID)) > 0
    )
);




RemoveIf(colToggleLog, CatItem_ID = ThisItem.CatItem_ID);
Collect(colToggleLog, { CatItem_ID: ThisItem.CatItem_ID, Desired: true })



RemoveIf(colToggleLog, CatItem_ID = ThisItem.CatItem_ID);
Collect(colToggleLog, { CatItem_ID: ThisItem.CatItem_ID, Desired: false })



