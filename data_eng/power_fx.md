// Rebuild working set for this module + this category
ClearCollect(
    colModuleCatItems_Working,
    AddColumns(
        Filter(
            Skill_Matrix_CategoryItems,
            Category = cmbSelectCategory.Selected.Value
        ),
        "WasSelectedBefore",
        !IsBlank(
            LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varModuleIdSafe &&
                CatItem_ID = CatItem_ID &&
                IsActive = true
            )
        ),
        "IsSelectedForModule",
        !IsBlank(
            LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varModuleIdSafe &&
                CatItem_ID = CatItem_ID &&
                IsActive = true
            )
        )
    )
);
