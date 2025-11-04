// 1) Cache the chosen module
Set(varSelectedModuleId, "" & cmbExistingModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbExistingModule.Selected.Title);

// 2) Snapshot ACTIVE refs for this module (normalized id)
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && IsActive = true
            ),
            CatItem_ID
        ),
        id, Text(CatItem_ID)
    )
);

// 3) If a category is already chosen, rebuild the working list for that category
If(
    !IsBlank(cmbSelectCategory.Selected),
    With(
        { catText: Text(cmbSelectCategory.Selected.Value) },
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Text(Category) = catText
                ),
                id, Text(CatItem_ID),
                IsSelectedForModule,
                    CountIf(colRefActiveIds, id = Text(CatItem_ID)) > 0
            )
        )
    ),
    // otherwise, clear the grid until a category is picked
    Clear(colModuleCatItems_Working)
);

// 4) Clear pending toggles/edits whenever module changes
Clear(colToggleLog);
Clear(colTypeEdits);
