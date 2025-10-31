// rebuild working list based on final saved state
If(
    And(
        !IsBlank(varSelectedModuleId),
        !IsBlank(cmbSelectCategory.Selected),
        !IsBlank(cmbSelectCategory.Selected.Value)
    ),
    With(
        {
            _catName: cmbSelectCategory.Selected.Value,
            _modId: varSelectedModuleId
        },
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    Category = _catName
                ),
                "WasSelectedBefore",
                !IsBlank(
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = _modId &&
                        CatItem_ID = ThisRecord.CatItem_ID &&
                        IsActive = true
                    )
                ),
                "IsSelectedForModule",
                !IsBlank(
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = _modId &&
                        CatItem_ID = ThisRecord.CatItem_ID &&
                        IsActive = true
                    )
                )
            )
        )
    ),
    // else: if no valid module/category, just clear
    Clear(colModuleCatItems_Working)
);
