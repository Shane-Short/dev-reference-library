// Rebuild working list after save to reflect final Reference state
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
    Clear(colModuleCatItems_Working)
);
