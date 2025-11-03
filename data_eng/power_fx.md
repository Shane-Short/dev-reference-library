// --- Final sync: refresh Reference and rebuild working set so checkboxes reflect truth ---
Refresh(Skill_Matrix_Reference);

// active refs for the current module (by CatItem_ID)
ClearCollect(
    colRefActiveIds,
    ShowColumns(
        Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
        "CatItem_ID"
    )
);

// Rebuild the category list the user is looking at, with prechecked flags.
// If no category is selected, clear the working set.
If(
    !IsBlank(varSelectedModuleId) && !IsBlank(cmbSelectCategory.Selected),
    ClearCollect(
        colModuleCatItems_Working,
        AddColumns(
            Filter(
                // normalize Category to text so Choice/Text both work
                AddColumns(Skill_Matrix_CategoryItems, CategoryText, Text(Category)),
                CategoryText = Text(cmbSelectCategory.Selected.Value)
            ),
            IsSelectedForModule,
            CountIf(colRefActiveIds, CatItem_ID = ThisRecord.CatItem_ID) > 0
        )
    ),
    Clear(colModuleCatItems_Working)
);
