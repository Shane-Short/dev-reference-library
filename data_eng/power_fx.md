// Guard: we need BOTH a module and a category before we build the working list
If(
    Or(
        IsBlank(varModuleIdSafe),
        IsBlank(cmbSelectCategory.Selected),
        IsBlank(cmbSelectCategory.Selected.Value)
    ),
    // If we're missing either, just clear the working collection and bail
    Clear(colModuleCatItems_Working),
    
    // Otherwise rebuild working set for this module+category
    With(
        {
            _catName: cmbSelectCategory.Selected.Value,
            _modId: varModuleIdSafe
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
    )
);








// when module changes, store module and clear downstream state
Set(varModuleIdSafe, cmbSelectModule.Selected.Mod_ID);
Set(varSelectedModuleName, cmbSelectModule.Selected.ModuleName); // or .Title depending on your schema

Reset(cmbSelectCategory);
Clear(colModuleCatItems_Working);   // <-- leave working list empty until a category is chosen







