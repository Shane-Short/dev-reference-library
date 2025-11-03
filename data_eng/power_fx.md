// Guard
If(
    Or(IsBlank(varSelectedModuleId), IsBlank(cmbSelectCategory.Selected), IsBlank(cmbSelectCategory.Selected.Value)),
    Clear(colModuleCatItems_Working),
    With(
        {
            _modId: varSelectedModuleId,
            _catRaw: Text(cmbSelectCategory.Selected.Value),
            // escape single quotes defensively (rarely needed, but safe)
            _catSafe: Substitute(Text(cmbSelectCategory.Selected.Value), "'", "''")
        },
        // 1) Pull all ACTIVE refs for this module once
        ClearCollect(
            colRefActiveIds,
            ShowColumns(
                Filter(
                    Skill_Matrix_Reference,
                    Mod_ID = _modId && IsActive = true
                ),
                "CatItem_ID"
            )
        );

        // 2) Pull the category’s items (coerce Choice/Text)
        //    If Category is a Choice, use Category.Value; if Text, use Category.
        ClearCollect(
            colModuleCatItems_Working,
            AddColumns(
                Filter(
                    Skill_Matrix_CategoryItems,
                    If(
                        IsRecord(Category),
                        Text(Category.Value),
                        Text(Category)
                    ) = _catRaw
                ),
                // Compute selected flag by checking the local set (no server calls here)
                "IsSelectedForModule",
                CountIf(
                    colRefActiveIds,
                    CatItem_ID = ThisRecord.CatItem_ID
                ) > 0
            )
        )
    )
);
