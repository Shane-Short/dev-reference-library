// when module changes, store module and clear downstream state
Set(varModuleIdSafe, cmbSelectModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbSelectModule.Selected.ModuleName); // or .Title depending on your schema

Reset(cmbSelectCategory);
Clear(colModuleCatItems_Working);   // <-- leave working list empty until a category is chosen


// Guard: require both module + category before we build the working list
If(
    Or(
        IsBlank(varSelectedModuleId),
        IsBlank(cmbSelectCategory.Selected),
        IsBlank(cmbSelectCategory.Selected.Value)
    ),
    // Missing module or category? Just clear and bail.
    Clear(colModuleCatItems_Working),

    // Otherwise rebuild working set for that module+category
    With(
        {
            _catName: cmbSelectCategory.Selected.Value,
            _modId: varSelectedModuleId
        },
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                // pull every CatItem in this category
                Filter(
                    Skill_Matrix_CategoryItems,
                    Category = _catName
                ),
                // store whether that CatItem is already active in Reference for this module
                "WasSelectedBefore",
                !IsBlank(
                    LookUp(
                        Skill_Matrix_Reference,
                        Mod_ID = _modId &&
                        CatItem_ID = ThisRecord.CatItem_ID &&
                        IsActive = true
                    )
                ),
                // mirror that into the live toggle state the checkbox will bind to
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
    )
);
