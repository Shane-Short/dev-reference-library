// Guard
If(
    Or(IsBlank(varSelectedModuleId), IsBlank(cmbSelectCategory.Selected), IsBlank(cmbSelectCategory.Selected.Value)),
    Clear(colModuleCatItems_Working),
    With(
        {
            _modId: varSelectedModuleId,
            _catRaw: Text(cmbSelectCategory.Selected.Value)
        },
        // 1) Active reference rows for this module (once)
        ClearCollect(
            colRefActiveIds,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = _modId && IsActive = true),
                "CatItem_ID"
            )
        );

        // 2) Category items, normalized Category → text via IfError
        ClearCollect(
            colModuleCatItems_Working,
            With(
                {
                    src: AddColumns(
                            Skill_Matrix_CategoryItems,
                            "CategoryText",
                            With({ c: Category }, IfError(Text(c.Value), Text(c)))
                         )
                },
                AddColumns(
                    Filter(src, CategoryText = _catRaw),
                    "IsSelectedForModule",
                    CountIf(colRefActiveIds, CatItem_ID = ThisRecord.CatItem_ID) > 0
                )
            )
        )
    )
);
